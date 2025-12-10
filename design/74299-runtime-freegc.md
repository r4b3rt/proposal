# Directly freeing user memory to reduce GC work

PJ Malloy ([@thepudds](https://github.com/thepudds)) \
January 2025

**Discussion** at https://go.dev/issue/74299. \
**CL stack** at https://go.dev/cl/673695.

## Introduction

This covers a design and initial implementation for a mechanism
to free user memory eagerly to the Go runtime. The freed memory can be reused
immediately in subsequent allocations without needing to wait for the GC cycle to progress.
The mechanism is implemented in a new runtime function, `runtime.freegc`, which is inserted
by the compiler in cases where it can prove the memory is no longer used.

A call to `runtime.freegc` records memory as dead and tracks it for
later reuse. The allocator is updated to be able to safely
reuse these heap objects, and care is taken so that `runtime.freegc` is able
to coexist and cooperate with the concurrently running garbage collector.

Updates to the compiler include automatically inserting calls
to `runtime.freegc` such as when the compiler proves heap allocated memory
does not live past a function return and therefore may be safely freed
on function exit.

The compiler is also updated to do a form of alias analysis so
that even escaping heap memory can be immediately freed once
it is dead in some cases, such as the intermediate memory for appends
of a non-aliased slice in a loop.

When enabled, `runtime.freegc` allows faster reuse of user memory,
reducing the allocation rate from the GC's perspective, and thus
reducing the frequency of GC cycles, total GC CPU usage, and how often
write barriers are enabled. It can also produce a more cache-friendly
allocation pattern for user code.

This design composes well with the compiler's historic ability to do
stack allocation of statically-known sizes by filling in gaps that
stack allocation cannot currently handle. It also composes well with
possible future improvements like:
 * [memory regions](https://go.dev/issue/70257), which can
   dynamically cast a wider net via a new user-facing API.
 * [stack allocations of dynamically-sized small slices](https://go.dev/cl/664299), which
   currently targets slices <= 32 bytes.
 * [escape analysis improvements for interface arguments](https://go.dev/issue/72036),
   which for example in `w.Write(b)` would allow `b` to be stack allocated, but which
   would be limited to only helping statically-sized slices unless there is an
   ability to hook into something like `runtime.freegc`. Other improvements also
   become more broadly applicable and hence more worthwhile if `runtime.freegc` exists.

There is also a reasonably rich group of direct follow-on targets for `runtime.freegc`
that likely become available, and it becomes another tool in the toolbox for
the compiler and runtime.

### Hand-written free lists

Today, users can implement their own free lists and achieve some of the memory reuse
benefits described here. However, `runtime.freegc` is implemented at a low level of
the runtime and has various advantages like being able to reuse memory across types
more safely and easily, more directly hook into what the GC is doing, more efficiently
track reusable objects, and even change the GC shape of reusable objects, which is
something even unsafe user code cannot do.

Updating the compiler translates to the reuse being both safer and more automated
than hand written code using a custom free list.

And of course, the goal is to speed up safe, idiomatic Go code without requiring
a variety of ad-hoc user written free lists.

## High-level approach

In an attempt to help triangulate on what is possible for the runtime
and compiler to do together to reduce how much work the GC must do,
this document proposes updates to:

1. **The runtime**: implement `runtime.freegc` and associated runtime entry-points.
Freed objects are recorded in one of two types of new free lists, and
are then immediately reusable for the next allocation in the same span class
via updates to the allocator. Challenges here include:
    * Safety, including in the face of the concurrent GC,
      conservative scanning, and time delays between an object being observed by the GC
      to when it is processed.
    * Performance, specifically without materially slowing down
      the normal allocation path when there is no reuse, making it fast on the reuse path,
      and being careful to not increase memory usage too much due to tracking
      additional information.

2. **The compiler**: automatically insert runtime calls to track and free slices
allocated via `make` when the compiler can prove it is safe to do so. Also implemented
is automatically inserting calls to free the intermediate memory
from repeated calls to `append` in a loop (without freeing the final slice) when it
can prove the memory is unaliased. This is done regardless of whether
the final slice escapes (for example, even if the final slice is returned from
the function that built the slice using `append` in a loop).

A sample benchmark included below shows `strings.Builder` operating
roughly 2x faster.

An experimental modification to `reflect` to use `runtime.freegc`
and then using that `reflect` with `json/v2` gave reported memory
allocation reductions of -43.7%, -32.9%, -21.9%, -22.0%, -1.0%
for the 5 official real-world unmarshalling benchmarks from
`go-json-experiment/jsonbench` by the authors of `json/v2`, covering
the CanadaGeometry through TwitterStatus datasets.

A separate CL updates the builtin maps to free backing map storage
during growth and split to free logically dead table memory that
is not held by map iteration.

Note: there is no current intent to modify the standard library to have
explicit calls to `runtime.freegc`, and of course such an ability
would never be exposed to end-user code. The `reflect` and other standard
library modifications were experiments that were helpful while designing
and implementing the runtime side prior to some of the compiler work.

Additional steps that are natural follow-ons (but not yet attempted) include:

4. Recognizing `append` in loops that start with a slice of unknown provenance,
such as a function parameter, but still be able to automatically
free intermediate memory starting with the second slice growth if the slice
does not otherwise become aliased.

5. Recognizing `append` in an iterator-based loop to be able to prove it is
safe to free unaliased intermediate memory, such as in the patterns
shown in `slices.Collect`.  (**Update**: this is now implemented but not yet in review.)

6. Possibly extend to freeing more than just slices. Slices are a natural
first step, but the large majority of the work implemented is independent of whether
the heap object represents a slice. For example, on function exit, we likely will be able to
free the table memory for non-escaping maps.

## Runtime API overview

There are currently four primary runtime entry-points with working implementations,
plus some additional variations on these that are not listed here. There are additional
details with more draft API doc towards the end of this document.

`freegc` is the primary entry-point for freeing user memory. It is emitted directly
by the compiler and also underlies the implementation of the other `free*` runtime
entry-points.

```go
func freegc(ptr unsafe.Pointer, size uintptr, noscan bool)
```

`growsliceNoAlias` is used during an `append`. It is like the existing `growslice`,
but only for the case where we know there are no aliases for
the backing memory of the slice, and hence we can free the old backing
memory as the slice is updated with new, larger backing memory. Importantly, this
works even if the slice escapes, such as if the resulting slice is returned
from a function. `growsliceNoAlias` does both the grow and the free.

 ```go
 func growsliceNoAlias(oldPtr *any, newLen, oldCap, num int, et *byte) (ary []any)
 ```

`makeslicetracked64` (and variations) allocates a heap object and records
or "tracks" the allocated object's pointer. `makeslicetracked64` is like
the existing `makeslice64`, but fills in a `trackedObjs` for later
use by `freegcTracked`. The compiler is responsible for reserving stack space
for a properly sized backing array for `trackedObj` based on how many objects
will be tracked, which gives the runtime the space it needs to write down
its tracking information. (An alternative not yet attempted might be to use
the standard makeslice64 and have compiler-generated code update the tracking
information.)

```go
func makeslicetracked64(et *_type, len64 int64, cap64 int64, trackedObjs *[]trackedObj) unsafe.Pointer
```

`freegcTracked` is automatically inserted by the compiler in cases where we know the
lifetime of a heap-allocated object is tied to a certain scope (usually a function),
which allows for tracked heap objects to be automatically freed as a scope is exited.
`freegcTracked` must be paired with functions like `makeslicetracked64`.

```go
func freegcTracked(trackedObjs *[]trackedObj)
```

An additional runtime API was temporarily used by some CLs as a
stand-in for `makeslicetracked64` for convenience, but there was no
plan to use it beyond helping to bootstrap some of the work:

```go
// TEMPORARY API.
func trackSlice(sl unsafe.Pointer, trackedObjs *[]trackedObj)
```

### Pointer liveness and `freegc` vs. `freegcTracked`

This section summarizes some differences between `freegc` and `freegcTracked`
and attempts to explain why `freegcTracked` exists at all.

First, note that `freegc` accepts an `unsafe.Pointer` and hence keeps the pointer
alive. It therefore could be a pessimization in some cases
(such as a long-lived function or function that straddles GC transitions)
if the caller does not call `freegc` before or roughly when
the liveness analysis of the compiler would otherwise have determined
the heap object is reclaimable by the GC.

For certain allocations or in certain loops, we directly insert `freegc`
via the compiler almost exactly at the point where the memory goes dead (or
similarly insert a layer on top of `freegc` like `growsliceNoAlias`).
`freegc` says that the memory is immediately available for reuse,
which is especially valuable in a loop or when reusing memory is otherwise
useful without waiting to exit a function.

However, we would be giving up on many valuable targets if we did not also
take advantage of escape analysis results (based on its current implementation and
based on some escape analysis modifications as part of this work).

Escape analysis today provides primarily function-scoped lifetime results.
If we merely added `freegc` to function exit points to free objects with function-scoped
lifetimes, that would likely be a pessimization if the object otherwise would have
been dead before the function exit.

This is part of the reason `freegcTracked` exists and takes a different approach
than `freegc`. Our tracking of an allocation does not keep the tracked memory alive,
therefore avoiding the liveness pessimization described above when used at scope
exit points. Another benefit of `freegcTracked` is that it frees multiple
objects at scope exit, which means fewer calls into the runtime and less emitted
code (including compared to if we were to instead have one `freegc`
per object at scope exit).

In addition, a given pointer today can go dead at multiple places in a function,
which would also add complexity if we were to directly use `freegc` automatically
wherever the pointer goes dead, in contrast to `freegcTracked` effectively being
a single clean up operation as the function exits.

Another difference is `freegcTracked` is essentially a two-step process -- first
there must be a tracked allocation, and later there is a `freegcTracked` call
that operates on those tracked allocation(s). In contrast, `freegc` can
operate on any heap allocation as long as we know it is safe to do so.

Summarizing `freegc` vs. `freegcTracked`:

```
  freegc                                    freegcTracked
  -------------------------------------     ---------------------------
  frees single heap object                  can free multiple heap objects

  lower-level primitive                     implemented on top of freegc

  operates on any heap allocation           paired with tracked allocation function

  call keeps pointer alive                  tracking does not keep pointer alive

  primarily intended for use the moment     primarily intended for use at
  a pointer is otherwise dead               scope exit
```

The initial implementation of tracking avoided keeping the pointer alive
via an approach of storing a `uintptr` on the stack along with a sequence number (which is
incremented as the GC cycle progresses) to then later know whether it is
still valid to use the stored `uintptr` to free its heap object as the scope is exited.

While reviewing this work, Keith Randall recently suggested an
improvement to the implementation of `freegcTracked` where the runtime zeros out
the tracked pointers on the stack as the GC cycle progresses, which is somewhat
similar in effect to a weak pointer. The GC-related portions of the runtime
would take a more active role now in the tracking, rather than the `freegcTracked` code
inferring what has happened based on the sequence number. This new approach might
touch more pieces of the code base, but it is better overall, and the intent is to now
implement this improvement. The general characteristics of `freegcTracked`
described above still apply (except now there is no sequence number), and most
of the associated implementation is unaffected.

There are more details on these functions at the bottom of this document,
including draft API doc.

## Compiler changes overview

### Analyzing aliases

The changes described in this subsection do not rely on `runtime.freegcTracked`,
but do rely on `runtime.freegc` and `runtime.growsliceNoAlias`.

As part of this work, the compiler has been updated to prove whether certain variables
are aliased or not, and to also examine `append` calls to see if it can prove
whether the slice is not aliased.

If so, the compiler automatically inserts a call to the new
runtime function `runtime.growsliceNoAlias` (briefly described above),
which frees the old backing memory after slice growth is complete
and the old storage is logically dead.

This works with multiple `append` calls for the same slice,
including inside loops, and the final slice memory can be
escaping, such as returned from the function. (In that case,
the final slice value is of course not freed.)

An example target is:

```go
    func f1(input []int) []int {
        var s []int
        for _, x := range input {
            s = append(s, g(x))
        }
        return s
    }
```

After proving that the slice `s` cannot be aliased in that loop,
the compiler converts that roughly to:

```go
    func f1(input []int) []int {
        var s []int
        for _, x := range input {
			// s cannot be aliased here.
			// The old backing array for s is freed inside growsliceNoAlias after growth.
			// The final slice is not freed. It is returned to the caller of f1.
			// Note: this is simplified from the actual signature for growsliceNoAlias.
            s = growsliceNoAlias(s, g(x))
        }
        return s
    }
```

Note that proving a slice is unaliased must be careful in the presence
of loops (or gotos that form unstructured loops). In this next example,
we do not treat `s` as entirely unaliased because it is aliased on
the second pass through the loop:

```go
var alias []int

func f2() {
        var s []int
        for i := range 10 {
                s = append(s, i)
                alias = s
        }
}
```

For some additional details on the overall approach, see the new
`src/cmd/compile/internal/escape/alias.go` file in
[CL 710015](https://go.dev/710015) that is part of this work (especially
the `(*aliasAnalysis).analyze` method).

By tracking aliasing and renamings across function inlinings (as
well as some support for recognizing whether a `goto` is an unstructured loop),
the compiler is also able to automatically free intermediate memory from appends
used with iterators, such as in:

```go
    s := slices.Collect(seq)
```

or

```go
    var s2 []int
    s2 = slices.AppendSeq(s2, seqA)
    s2 = slices.AppendSeq(s2, seqB)
	s2 = slices.AppendSeq(s2, seqC)
```

(An earlier version of this work was only able to handle these iterator cases
via manual edits to the `slices` package; that is no longer needed).

### Analyzing `make`

The changes described in this subsection do rely on `runtime.freegcTracked`.

Additional CLs update the compiler to automatically insert free calls for slices allocated via
`make` when the compiler can prove it is safe to do so.

Consider for example the following code:

```go
func f1(size int) {
	s := make([]int64, size) // heap allocated if > 32 bytes
	_ = s
}
```

In today's compiler, the backing array for `s` is not marked as escaping,
but will be heap allocated if larger than 32 bytes.

As part of this work, the compiler has been updated so that the
backing array for `s` is recognized as non-escaping but unknown size and
it is valid to free it when `f1` is done.

The compiler automatically transforms `f1` to roughly:

```go
func f1(size int) {
	var freeablesArr [1]struct { // compiler counts how many are needed (one, in this case)
		ptr      uintptr
		// Possibly 1-2 additional fields in future.
	}
	freeables := freeablesArr[:0]
	defer freegcTracked(&freeables)

	s := make([]int64, size)
	trackSlice(unsafe.Pointer(&s), &freeables) // NOTE: only temporarily two runtime calls

	_ = s
}
```

TODO: remove the `trackSlice` from this example, which was only a temporary API used to help
bootstrap this work.

TODO: mention `make`-like uses of `append`, such as `append([]byte(nil), make([]byte, n)...)`.

The `freeablesArr` is stored on the stack and automatically sized by the compiler based
on the number of target sites. The `defer` is currently inserted in a way so that it is
an open-coded `defer` and hence has low performance impact, though the plan is to switch
to inserting the calls without relying on `defer`.

For more details on `freegcTracked`, see the high-level description above or the API doc below.

The transformed code just above shows what is currently implemented, but two items seen
there that might change as the implementation is polished:
 * `freegcTracked` currently takes a slice for convenience, but we could make that tighter.
 * the current intent is to use `makeslicetracked64` (for example, in `f1` above),
 but the compiler initially inserted calls to a new `runtime.trackSlice` instead,
 which initially seemed slightly easier to implement on the compiler side.
 The plan is to have the compiler use `makeslicetracked64` instead.

If there are multiple target sites within a function, such as in `f2` here:

```go
func f2(size1 int, size2 int, x bool) {
	s1 := make([]int64, size1) // heap allocated if > 32 bytes
	if x {
		s2 := make([]int64, size2) // heap allocated if > 32 bytes
		_ = s2
	}
	_ = s1
}
```

The compiler counts the max number of sites (not sensitive to conditionals),
and sizes `freeablesArr` appropriately (two element array in this case),
with a single call to `freegcTracked` per function:

```go
func f2(size1 int, size2 int, x bool) {
	var freeablesArr [2]struct { // compiler counts how many are needed (two, in this case)
		ptr      uintptr
		// Possibly 1-2 additional fields in future.
	}
	freeables := freeablesArr[:0]
	defer freegcTracked(&freeables) // only a single freegcTracked call, regardless of how many tracked sites

	s1 := make([]int64, size1)
	trackSlice(unsafe.Pointer(&s1), &freeables)
	if x {
		s2 := make([]int64, size2)
		trackSlice(unsafe.Pointer(&s2), &freeables) // NOTE: only temporarily two runtime calls

		_ = s2
	}
	_ = s1
}
```

TODO: remove the `trackSlice` from this example, which was only a temporary API used to help
bootstrap this work.

The max number of automatically inserted call sites is bounded
based on the count of `make` calls in a given function.

Inside `for` loops or when a `goto` looks like it is
being used in an unstructured loop, the compiler
currently skips automatically inserting these calls.

However, a modest extension (and the current plan) is to have the compiler
instead use a different runtime entry point for freeing in loops. The idea is
to allow heap-allocated slice memory to be reclaimed on each loop iteration
when it returns to the same allocation site, and when it's safe to do so.
Because it's the same allocation site, this doesn't require unbounded
or dynamic space in the trackedObj slice.  (The compiler can already prove
it is safe and the runtime entry point is prototyped.)

## `strings.Builder`

As an early experiment, we updated `strings.Builder` to have manual calls
to `runtime.freegc`, though there is no intent to actually manually
modify `strings.Builder`.

`strings.Builder` is a somewhat
low-level part of the stdlib that already helps encapsulate some
performance-related trickery, including using `unsafe` and
`internal/bytealg.MakeNoZero`. The `strings.Builder` already owns its in-progress
buffer that is being built up, until it returns a usually final string.
The existing internal comment says:

```go
    // External users should never get direct access to this buffer, since
    // the slice at some point will be converted to a string using unsafe,
    // also data between len(buf) and cap(buf) might be uninitialized.
    buf []byte
```

If we update `strings.Builder` to explicitly call `runtime.freegc` on
that `buf` after a resize operation (but without freeing the usually final
incarnation of `buf` that will be returned to the user as a string via `Builder.String`), we
can see some nice benchmark results on the existing strings benchmarks
that call `strings.(*Builder).Write` N times and then call `String`.  Here, the
(uncommon) case of a single `Write` is not helped (given it never
resizes after first alloc if there is only one `Write`), but the impact grows
such that it is up to ~2x faster as there are more resize operations due
to more `Write` calls:

```
                                               │     disabled.out │      new-free-20.txt                │
                                               │      sec/op      │   sec/op     vs base                │
BuildString_Builder/1Write_36Bytes_NoGrow-4           55.82n ± 2%   55.86n ± 2%        ~ (p=0.794 n=20)
BuildString_Builder/2Write_36Bytes_NoGrow-4           125.2n ± 2%   115.4n ± 1%   -7.86% (p=0.000 n=20)
BuildString_Builder/3Write_36Bytes_NoGrow-4           224.0n ± 1%   188.2n ± 2%  -16.00% (p=0.000 n=20)
BuildString_Builder/5Write_36Bytes_NoGrow-4           239.1n ± 9%   205.1n ± 1%  -14.20% (p=0.000 n=20)
BuildString_Builder/8Write_36Bytes_NoGrow-4           422.8n ± 3%   325.4n ± 1%  -23.04% (p=0.000 n=20)
BuildString_Builder/10Write_36Bytes_NoGrow-4          436.9n ± 2%   342.3n ± 1%  -21.64% (p=0.000 n=20)
BuildString_Builder/100Write_36Bytes_NoGrow-4         4.403µ ± 1%   2.381µ ± 2%  -45.91% (p=0.000 n=20)
BuildString_Builder/1000Write_36Bytes_NoGrow-4        48.28µ ± 2%   21.38µ ± 2%  -55.71% (p=0.000 n=20)
```

There is no plan to hand insert calls to `runtime.freegc` within `strings.Builder`
(including there are alternatives that could be used within
`strings.Builder`, such as a segmented buffer and/or `sync.Pool`
such as used within `cmd/compile`). We are mainly using it here as a
small-ish concrete example.

In addition, after doing the hand edited `strings.Builder` prototype,
the compiler work has since progressed further, and I have some hope for
the compiler to be able to automatically free intermediate memory
for many common uses of `strings.Builder` via `growsliceNoAlias`
(with roughly 2 of 3 steps towards that goal currently implemented;
`strings.Builder` is more complex than an ordinary use of `append`
for a few reasons).

There are other potential optimizations for `strings.Builder`,
like allocating an initial buf (e.g., 64 bytes)
on the first `Write`, faster growth than current 1.25x growth for larger
slices, and "right sizing" the final string; an earlier cut did
implement those optimizations.

## Normal allocation performance

For the cases where a normal allocation is happening without any reuse,
some initial micro-benchmarks suggest the impact of these changes could
be small to negligible (at least with GOAMD64=v3):

```
goos: linux
goarch: amd64
pkg: runtime
cpu: AMD EPYC 7B13
                        │ base-512M-v3.bench │   ps16-512M-goamd64-v3.bench     │
                        │        sec/op      │ sec/op     vs base               │
Malloc8-16                      11.01n ± 1%   10.94n ± 1%  -0.68% (p=0.038 n=20)
Malloc16-16                     17.15n ± 1%   17.05n ± 0%  -0.55% (p=0.007 n=20)
Malloc32-16                     18.65n ± 1%   18.42n ± 0%  -1.26% (p=0.000 n=20)
MallocTypeInfo8-16              18.63n ± 0%   18.36n ± 0%  -1.45% (p=0.000 n=20)
MallocTypeInfo16-16             22.32n ± 0%   22.65n ± 0%  +1.50% (p=0.000 n=20)
MallocTypeInfo32-16             23.37n ± 0%   23.89n ± 0%  +2.23% (p=0.000 n=20)
geomean                         18.02n        18.01n       -0.05%
```

## runtime API

This section lists the runtime APIs that are automatically inserted by the compiler.

Note: the term "logically dead" here is distinct from the compiler's liveness
analysis, and the runtime free and re-use implementations must be (and
attempt to be) robust in sequences such as a first alloc of a given
pointer, GC observes and queues the pointer, runtime.free, GC dequeues
pointer for scanning, runtime re-uses the same pointer, all within a
single GC phase. (TODO: more precise term than "logically dead"?)

```go
// freegc records that a heap object is reusable and available for
// immediate reuse in a subsequent mallocgc allocation, without
// needing to wait for the GC cycle to progress.
//
// The information is recorded in a free list stored in the
// current P's mcache. The caller must pass in the user size
// and whether the object has pointers, which allows a faster free
// operation.
//
// freegc must be called by the effective owner of ptr who knows
// the pointer is logically dead, with no possible aliases that might
// be used past that moment. In other words, ptr must be the
// last and only pointer to its referent.
//
// The intended callers are the runtime and compiler-generated calls.
//
// ptr must point to a heap object or into the current g's stack,
// in which case freegc is a no-op. In particular, ptr must not point
// to memory in the data or bss sections, which is partially enforced.
// For objects with a malloc header, ptr should point mallocHeaderSize bytes
// past the base; otherwise, ptr should point to the base of the heap object.
// In other words, ptr should be the same pointer that was returned by mallocgc.
//
// In addition, the caller must know that ptr's object has no specials, such
// as might have been created by a call to SetFinalizer or AddCleanup.
// (Internally, the runtime deals appropriately with internally-created
// specials, such as specials for memory profiling).
//
// If the size of ptr's object is less than 16 bytes or greater than
// 32KiB - gc.MallocHeaderSize bytes, freegc is currently a no-op. It must only
// be called in alloc-safe places.
//
// Note that freegc accepts an unsafe.Pointer and hence keeps the pointer
// alive. It therefore could be a pessimization in some cases (such
// as a long-lived function) if the caller does not call freegc before
// or roughly when the liveness analysis of the compiler
// would otherwise have determined ptr's object is reclaimable by the GC.
func freegc(ptr unsafe.Pointer, size uintptr, noscan bool) bool

// growsliceNoAlias is like growslice but only for the case where
// we know that oldPtr is not aliased.
//
// In other words, the caller must know that there are no other references
// to the backing memory of the slice being grown aside from the slice header
// that will be updated with new backing memory when growsliceNoAlias
// returns, and therefore oldPtr must be the only pointer to its referent
// aside from the slice header updated by the returned slice.
func growsliceNoAlias(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice

// makeslicetracked64 is like the existing makeslice64, but the caller
// also passes in a pointer to a trackedObj slice that allows the runtime
// to track the heap-allocated backing array and possibly later
// free the tracked objects via freegcTracked. The contents of
// the trackedObj slice are opaque to the caller, but the caller
// provides the space for the trackedObj slice.
//
// Currently, the only intended caller is code inserted by
// the compiler, which only does so when escape analysis can
// prove the associated pointer has a known scope -- currently,
// when it can prove the pointer does not outlive the current
// function. This effectively creates a scope for the
// associated heap pointers. The compiler also arranges for
// freegcTracked to be called on function exit, so that
// the heap objects are automatically freed when leaving the
// scope. The compiler also arranges for the trackedObj slice's
// backing array to be placed on the user's normal goroutine
// stack to provide storage for the tracking information,
// with that storage matching the lifetime of the tracked
// heap objects (that is, both match the lifetime of
// the function call).
func makeslicetracked64(et *_type, len64 int64, cap64 int64, trackedObjs *[]trackedObj) unsafe.Pointer

// freegcTracked marks as reusable the objects allocated by
// calls to makeslicetracked64 and tracked in the trackedObjs
// slice, unless the GC has already possibly reclaimed
// those objects.
//
// The trackedObj slice does not keep the associated pointers
// alive past the start of a mark cycle, and
// freegcTracked becomes a no-op if the GC cycle
// has progressed such that the GC might have already reclaimed
// the tracked pointers before freegcTracked was called.
// This might happen for example if a function call lasts
// multiple GC cycles.
//
// The compiler statically counts the maximum number of
// possible makeslicetracked64 calls in a function
// in order to statically know how much user
// stack space is needed for the trackedObj slice.
func freegcTracked(trackedObjs *[]trackedObj)

// trackedObj is used to track a heap-allocated object that
// has a proven scope and can potentially be freed by freegcTracked.
//
// trackedObj does not keep the tracked object alive.
//
// trackedObj is opaque outside the allocation and free code.
//
// TODO: update description to describe runtime zeroing.
//
// TODO: there is a plan to allow a trackedObj to have specials,
// including user-created via SetFinalizer or AddCleanup, but
// not yet implemented. (In short -- update runtime.addspecial
// to update a bitmap of objIndexes that have specials so
// that the fast path for allocation & free does not need to
// walk an mspan's specials linked list, and then a trackedObj with
// specials will not be reused. An alternative might be to
// sweep the span and process the specials after reaching
// the end of an mspan in the mcache if there are reusable
// objects with specials, but that might be harder or perhaps
// infeasible. More generally, sweeping sooner than the
// normal GC but not in the allocation/free fast path (perhaps
// in a goroutine managed by the runtime), might be worth
// exploring for other reasons than just specials).
type trackedObj struct {
	ptr      uintptr   // tracked object
	// Future: possibly 1-2 additional metadata fields to speed up free.
}
```

## Other possible runtime APIs

### runtime.free without size, noscan

An earlier version of this work also implemented an API that just took an
`unsafe.Pointer`, without any size or noscan parameters, like:

```go
func free(ptr unsafe.Pointer)
```

However, this was more expensive, primarily due to needing to call
`spanOf`, `findObject` or similar. Early measurements suggested that
passing the size and noscan parameters like in `freegc` allowed
eliminating roughly 60% of the cost of a free operation.

(A small historical note: the more explicit API taking size and noscan parameters
was originally named `freeSized` but was later renamed to `freegc`; the older `freeSized` name
might still be visible in some earlier prototype CLs.)

### runtime.freegcSlice

Separately, we will likely also use a `runtime.freegcSlice`. It is a
small function over `runtime.freegc` that accepts a slice. (Initially it was
just a test function, but ended up being more convenient in some cases,
including in some of the early prototype compiler work.)
