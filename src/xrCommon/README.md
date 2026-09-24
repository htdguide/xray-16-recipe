# src/xrCommon — the standard library this project decided to have

Chapter 2 of [the build order](../../SYSTEM-REQUIREMENTS.md#7-build-order). Fifteen small
headers, about three hundred lines of source in total, and the highest ratio of decision to
code anywhere in the repository.

## What this module is responsible for

Every container the engine instantiates, every owning handle it constructs, and every
comparison it sorts by are named here rather than taken from the host language directly.
The module owns four things: **which allocator all storage comes from**, **which ordering
text is compared under**, **which tolerances two numbers are the same within**, and **what
an angle means**. It owns no behaviour of its own — there is no algorithm in this directory
except the angle arithmetic and one substring replacement — and it depends on almost
nothing.

It is also the chapter where a rebuilder is most likely to go wrong by doing the obvious
thing. Most of these files are, in the original, a single line that renames a standard
container. A rebuild in a language with its own containers deletes nine of the fifteen
outright and is correct to. The danger is deleting the *reason* along with the file: the
reason is a contract the rest of the engine was written against, and it does not announce
itself anywhere else.

## Where it sits

It rests on chapter 1 ([`src/Common`](../Common/README.md)) for the compiler and platform
names, and — awkwardly, since chapter 6 is later — it reaches forward to
[`src/xrCore`](../xrCore/README.md) for three things: the allocator implementation, the
interned-string type, and the numeric constants. That forward reach is a genuine cycle in
the source's own layering, and it is why this chapter is described here rather than folded
into chapter 6: these headers are what *everything else* includes, and pulling the whole of
chapter 6 in with them would make every file in the engine depend on the filesystem.

A rebuild dissolves the awkwardness by putting the allocator, the string type and the
constants in this module and letting chapter 6 depend on it, which is the layering the
source clearly wanted.

Everything from chapter 3 onward uses these names.

## The four ideas, stated once

The twins are terse because these four ideas are stated here.

**1. One heap.** Every allocation in the process comes from one allocator and returns to
it. That allocator is stateless — any two instances are interchangeable, so containers move
and swap in constant time — aligns every block to at least sixteen bytes so that four-wide
float values can be stored in ordinary records, and accepts a free that names only the
address. It also has *no policy for running out of memory*: allocation failure returns
nothing and nothing checks. Full contract in [`xr_allocator.h`](xr_allocator.h.md).

**2. Two text types, and the split is deliberate.** Text the engine *stores and compares*
is immutable, reference-counted and de-duplicated in a process-wide intern table, so that
two names are equal in one pointer comparison and hash in one field read — this is the type
in [`xrCore/xrstring.h`](../xrCore/xrstring.h.md) and it is what almost every name in the
engine is. Text the engine *builds* is a mutable byte buffer, [`xr_string`](xr_string.h.md),
used to assemble a value and then interned. Both hold bytes in a single-byte codepage, not
Unicode; the translation happens at the text-table reader and nowhere else.

**3. Case folding is ASCII-only, everywhere, by requirement.** The shipped game data
references files and sections with inconsistent capitalisation, so lookups fold case —
but the fold covers exactly the twenty-six unaccented Latin letters and nothing else. A
locale-aware fold would make the set of files a level matches depend on the user's system
language. The two orderings that implement this are the whole content of
[`predicates.h`](predicates.h.md), and the same rule governs the in-place fold in
[`xr_string.h`](xr_string.h.md).

**4. Nothing compares reals for equality.** There are three absolute tolerances — one for
"is this zero", one for "are these the same value", one for "is this the same place in the
world" — and picking the right one is a decision at every call site. They are absolute, not
relative, and tuned against level geometry authored in metres. Named in
[`math_funcs_inline.h`](math_funcs_inline.h.md).

To which add the one piece of real arithmetic in the chapter: **an angle is a bare number of
radians with no type**, so the range it is in, and what "the difference between two angles"
means, are decided per call site out of the vocabulary in
[`math_funcs.h`](math_funcs.h.md). Every creature that turns the short way round turns
because of that file.

## The twins

| File | Role |
|---|---|
| [`xr_allocator.h`](xr_allocator.h.md) | **The allocation contract** every other file here defers to. Substantive. |
| [`predicates.h`](predicates.h.md) | **The two text orderings** — exact, and ASCII case-folded. Substantive. |
| [`xr_string.h`](xr_string.h.md) | **Mutable text**: the buffer you build in before interning. Case folding, hashing, replace-all. Substantive. |
| [`math_funcs.h`](math_funcs.h.md) | **Angle arithmetic**: normalization ranges, shortest-way-round difference, the two rate-limited follow loops. Substantive; bodies live in [`utils/xrMiscMath/vector.cpp`](../utils/xrMiscMath/vector.cpp.md). |
| [`math_funcs_inline.h`](math_funcs_inline.h.md) | **The three tolerances**, clamping, degree conversion, grid snapping. Substantive. |
| [`xr_smart_pointers.h`](xr_smart_pointers.h.md) | Owning handles that release through the engine allocator; the deterministic-destruction requirement. Substantive. |
| [`misc_math_types.h`](misc_math_types.h.md) | An orientation as yaw, pitch and roll — and the composition order nobody wrote down. |
| [`xr_array.h`](xr_array.h.md) | Fixed-size array; plus a padding fossil preserving a record size with no surviving reader. |
| [`xr_vector.h`](xr_vector.h.md) | Growable contiguous array — the one whose storage address is handed to the graphics device. |
| [`xr_deque.h`](xr_deque.h.md) | Double-ended queue; chosen where element addresses must survive appends. |
| [`xr_list.h`](xr_list.h.md) | Linked sequence; chosen where positions survive mutation during iteration. |
| [`xr_map.h`](xr_map.h.md) | Ordered key-to-value table, single- and multi-key. Iteration order is observable output. |
| [`xr_set.h`](xr_set.h.md) | Ordered key collection; membership is decided by the ordering, which is how case-insensitive sets happen. |
| [`xr_unordered_map.h`](xr_unordered_map.h.md) | Hashed table; iteration order is never observable output. |
| [`xr_stack.h`](xr_stack.h.md) | Last-in-first-out over the contiguous array rather than the default queue — a real choice, for the per-frame descent stacks. |

## Note on the declaration shorthands

Eight of these files define abbreviations for declaring a named container type together
with its iterator type in one line. They appear in the thousands across the engine and mean
nothing whatsoever; they exist because the original language requires an iterator type to
be named separately and the authors got tired of it. A rebuild deletes them without reading
them. The one exception worth noticing is the ordered table's and ordered collection's
*explicit-ordering* forms: their presence at a declaration is the marker that this table
chose a non-default ordering, and is the fastest way to find every case-insensitive
structure in the engine.
