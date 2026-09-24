# src/xrCore/_rect.h

> The rectangle: four numbers named both as corners and as edges, in a real and an integer flavour — the screen-space region type the whole UI and the render target code are written against.

**Needs** — [`_vector2.h`](_vector2.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`../xrCommon/math_funcs_inline.h`](../xrCommon/math_funcs_inline.h.md)

**Used by** — [`_stl_extensions.h`](_stl_extensions.h.md) · [`vector.h`](vector.h.md) · [`xrCore.h`](xrCore.h.md)

**Tier floor** — T1: four contiguous numbers read and written as a memory image, handed to the graphics device as a scissor rectangle and parsed out of the UI layout files in that order.

## Purpose

Every widget's extent, every scissor rectangle, every texture atlas region and every
screen-space viewport is one of these. It differs from the 2D box in
[`_fbox2.h`](_fbox2.h.md) in vocabulary rather than in substance: this one names its fields
left/top/right/bottom, offers intersection and clipping, and comes in an integer flavour for
pixel work; that one names them minimum/maximum, offers accumulation, and is reals only.
The two overlap heavily and a rebuild should have one type — see the
[note in the 2D box](_fbox2.h.md#purpose).

The type is generic over its component, instantiated for 32-bit real and 32-bit signed
integer. The integer flavour is the one that matters most: pixel rectangles must not drift
through real arithmetic, and the scissor rectangle the graphics device takes is integral.

## State

```text
RECORD Rect of T
  x1 : T     # also named: left
  y1 : T     #             top
  x2 : T     #             right
  y2 : T     #             bottom
```

Four components in that order, addressable four ways over one set of bytes — as numbered
corners, as named edges, as two 2-vectors (top-left and bottom-right), and as a flat array of
four. The multiple names exist because the UI layout files speak of edges, the geometry code
speaks of corners, and the device speaks of an array.

**Invariants**

- **The vertical axis points down.** `top` is the smaller value and `bottom` the larger,
  which is the screen convention and the opposite of the world convention the 3D box uses.
  Every comparison here reads correctly only under it.
- The **invalid** rectangle sets the top-left to the largest representable value and the
  bottom-right to the smallest, the same accumulation identity the boxes use. It is also what
  "empty" means for this type — the two are one value and one operation under two names.
- A rectangle is **valid** when the left edge is strictly less than the right and the top
  strictly less than the bottom. Strictly: a zero-width or zero-height rectangle is *empty*,
  which is the opposite of the boxes' convention, where a degenerate box is valid. That
  difference is real and is what a clipping type wants — an empty clip must reject everything
  — while an accumulating type wants the degenerate case to be legal. See the note under
  [Validity](#validity): in the original this predicate is written but never reachable.

## Construction and reset

**Contract** — Set from four components, from two corner vectors, from another rectangle; to
all zeros; to the accumulation identity (which is also "set empty"). All allocation-free,
all returning the rectangle so calls chain.

## The arithmetic family

**Contract** — Add, subtract, multiply and divide, each by a separate horizontal and vertical
amount, each in an in-place form and a form reading from another rectangle. Every one applies
to **all four** components.

**Notes** — Applying the multiply to all four components scales *about the origin*, not about
the rectangle's own centre: a rectangle far from the origin moves as well as grows. That is
what a coordinate-space conversion wants — pixels to normalized device coordinates, say — and
is not what "make this 10% bigger" wants. The grow and shrink operations below are the second
of those.

There is no division guard anywhere. On the integer flavour these are integer operations and
truncate, which is what pixel work needs and is a trap for anyone who expects rounding.

## `grow` and `shrink`

**Contract** — Push the edges apart or pull them together by a horizontal and a vertical
amount, leaving the centre where it is. A shrink larger than half the size inverts the
rectangle and is not checked — the result is an invalid rectangle, which the validity
predicate will correctly reject.

## `in` — containment

**Contract** — Whether a point lies within, inclusive on all four edges. Available for a pair
of components and for a 2-vector.

**Notes** — Inclusive on all four edges means a point on the shared edge of two adjacent
rectangles is in both. For hit-testing a UI that is harmless (the widget tree resolves it by
order); for tiling a texture atlas it is the reason a one-pixel gutter is needed.

## `cmp` — comparison

**Contract** — Two forms: the integer rectangle compares exactly, the real rectangle
compares componentwise within the loose epsilon. Which form is reached depends on the type
of the argument, not the type of the rectangle.

**Notes** — Both forms are declared on both flavours, which means an integer rectangle can be
compared exactly against another integer rectangle and approximately against a real one. That
is an artifact of how the generic is written rather than a decision; a rebuild gives each
flavour its own comparison.

## `intersected` and `intersection`

**Contract** — Two operations that belong together:

- **intersected** — whether two rectangles overlap, by four separating comparisons. Available
  as a test between two given rectangles and as a test of this one against another.
  Touching edges count as overlapping, because the comparisons are strict.
- **intersection** — writes the overlap of two rectangles into this one and reports whether
  there was one; leaves this rectangle untouched when there is not. Componentwise maximum of
  the two top-lefts and minimum of the two bottom-rights.

**Invariants** — The intersection is only written when the test passes, so a caller that
ignores the result reads a stale rectangle rather than an empty one. Given that this type's
empty value is well defined, a rebuild should write the empty rectangle on failure and drop
the boolean — the current shape is a source of clipping bugs that only appear when a widget
scrolls fully off screen.

**Notes** — This pair is the whole reason the type exists separately from the 2D box: it is
the clip operation, and a widget tree's layout pass is a recursive intersection of a child's
extent with its parent's clip.

## Measurement

**Contract** — Centre, size as a vector, width and height individually. All read-only.

**Notes** — The centre is computed as the sum of the two corners divided by two, which on
the integer flavour truncates — so the centre of a rectangle with an odd width sits one pixel
left of true. Every UI element centred this way is consistently off by that half pixel, which
is invisible and is worth knowing before a rebuild "fixes" it and moves every centred widget.

## Validity and emptiness

**Contract** — Two different questions with confusingly similar names in the original: one
asks whether the four numbers are representable (finite and normal), the other whether the
rectangle encloses anything. A rebuild should name them apart, and should provide both — a
clip pass needs the second every frame.

**Notes** — **Neither predicate is reachable in the original, and both are wrong as
written.** The emptiness predicate reads a field name that does not exist on the corner
vector, and the representability predicate calls a member that does not exist either; both
compile only because the type is generic and its unused operations are never instantiated.
No caller in the tree reaches either. So there is no behaviour to preserve here, only the
intent — which is the one stated above.

This is worth carrying as a general warning about the type: it is generic, so anything not
exercised by a caller has never been compiled, and a rebuild that instantiates the whole
surface will find more than these two.
