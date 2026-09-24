# src/xrPhysics/PHItemList.h

> A singly-linked list whose links live inside the elements, so an object can remove itself from the world's active list in constant time without knowing its position.

**Needs** — _(none beyond the core containers)_
**Used by** — [`PHObject.h`](PHObject.h.md) · [`PHUpdateObject.h`](PHUpdateObject.h.md)
**Tier floor** — T2: a container with an intrusive-link requirement; expressible anywhere that allows a record to hold a link to its own slot.

## Purpose

Every physics object belongs to exactly one of several world-level lists (active, frozen,
recently-disabled) and moves between them constantly — a crate falls asleep, is bumped awake, is
frozen while another object is resolved. The engine needs two operations to be O(1): *erase this
element* and *move an entire list into another list*. A list that stores its elements by value
gives neither.

The structure is a forward list where each element additionally stores a link to the slot that
points at it. Erasing is then a write through that link; nothing is searched.

## State

```text
RECORD ListLinks                     # embedded in every element type that can be listed
  next     : optional<Element>
  back     : handle to the slot that holds this element
                                     # invariant: back always names either the list's head slot
                                     #            or the previous element's `next` slot

RECORD ItemList<Element>
  head     : optional<Element>
  tail     : handle to the slot a new element will be written into
  size     : int (16-bit)
                                     # invariant: tail names `head` when size = 0,
                                     #            otherwise the last element's `next` slot
                                     # invariant: size never reaches its 16-bit ceiling; the
                                     #            engine asserts rather than growing the counter,
                                     #            because a level that needs more than ~65k
                                     #            simultaneously-simulated objects is a bug
```

## `push_back`

**Contract** — appends in constant time. Writes the element through the tail slot, records the
reverse link in the element, advances the tail to the element's own `next` slot, and terminates it.

## `erase`

**Contract** — removes an element in constant time given only the element, not its position. Writes
the element's successor through the element's back-link; if there was no successor, the list's tail
slot becomes the back-link, which is what keeps `push_back` correct afterwards.

```text
FUNCTION erase(element)
  REQUIRE size > 0
  successor := element.next
  write successor through element.back
  IF successor EXISTS THEN successor.back := element.back
  ELSE tail := element.back
  size := size - 1
```

## `move_items`

**Contract** — splices an entire source list onto the end of this one and empties the source, in
constant time regardless of length. This is how the world freezes and thaws everything at once.

```text
FUNCTION move_items(source)
  IF source IS EMPTY THEN RETURN
  write source.head through tail
  source.head.back := tail
  tail  := source.tail
  size  := size + source.size
  source.clear()
```

**Notes** — `clear` does not visit elements. Their embedded links become stale and are rewritten on
the next `push_back`. A rebuild that keeps the links valid must not pay a traversal here, or
freezing a busy level becomes a per-frame cost.

## `ItemStack`

**Contract** — the same list, except each element additionally records its index at insertion time.
Used where an element must be able to name its own position in the collection it was added to;
nothing renumbers on erase, so the recorded index is only meaningful for a stack that is built and
torn down as a unit.
