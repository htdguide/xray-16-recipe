# src/editors/xrWeatherEditor/property_integer_values_value_getter.cpp

> A whole number indexing a list that only exists while the editor is running, so the list is asked for every time.

**Needs** — [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md) · [`property_integer.hpp`](property_integer.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md); callers name that, not this file.
**Tier floor** — T2: owns native callback objects and rebuilds a managed list from native text on every query

## Purpose

The live variant of [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md). Used where the set of choices is discovered rather than declared — what weather sets exist on disk, what keyframes the current set has — and can change while the row is on screen.

## State

```text
RECORD LiveIndexSelectionProperty EXTENDS IntegerProperty
  list_getter : callback() -> native array of text   # owned copy
  list_size   : callback() -> int                    # owned copy
                                                     # invariant: the two agree —
                                                     # size describes that same array
```

There is no cached list. That is the whole point of the type: a cache is exactly what would go stale.

## `construct(getter, setter, list_getter, list_size)` · `release`

**Contract** — Takes private copies of both list callbacks alongside the base's value callbacks; frees all four exactly once. Release rule as in [`property_float.cpp`](property_float.cpp.md).

## `collection`

**Contract** — Asks the engine for the array and its length, copies every entry into a fresh managed list, and returns it. Allocates a new list on every call. Does not block. Must be called on the user-interface thread while the engine is not mutating the underlying set.

```text
FUNCTION collection() -> list<text>
  values = list_getter()
  result = empty list
  FOR EACH i IN 0 .. list_size() - 1
    result.append(copy(values[i]))
  RETURN result
```

**Notes** — The array's length is asked for separately rather than carried with the array, because the engine's side of this is a raw run of text pointers with no terminator. The two callbacks must describe the same array at the same instant; there is no guard, and the engine satisfies it by answering both from the same underlying container.

Copying every entry rather than holding the engine's text is not defensive tidiness — the array can be rebuilt by the engine between calls, so anything retained past the return is a dangling reference.

## `GetValue`

**Contract** — Reads the stored index and clamps it against the list's *current* length, which means rebuilding the list to find that length.

**Notes** — Rebuilding the whole list to learn its size is the cost of having no cache, and it is paid on every repaint of the row. It is acceptable because these lists are short and the editor is not frame-bound; a rebuild that finds otherwise should ask the engine for the size alone.

The clamp is doing real work here, unlike in the fixed-list case: the list genuinely shrinks under the row when the user deletes a keyframe, and the selection must land somewhere.

## `SetValue`

**Contract** — Rebuilds the list, finds the picked label's position, stores it. A label absent from the freshly rebuilt list is a programming error and is not handled — which is the one place the live variant is weaker than the fixed one, since the list *can* legitimately change between the grid offering a label and the user picking it.
