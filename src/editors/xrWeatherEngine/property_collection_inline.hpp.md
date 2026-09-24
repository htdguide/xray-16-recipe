# src/editors/xrWeatherEngine/property_collection_inline.hpp

> How an editable list behaves: what the grid may do to it, who owns the elements, and how a new element gets a name nothing else has.

**Needs** — [`property_collection.hpp`](property_collection.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_collection.hpp`](property_collection.hpp.md)
**Tier floor** — T2: it destroys elements at defined points, and the grid's rows are live references to them.

## Purpose

The substance behind [`property_collection.hpp`](property_collection.hpp.md). Two
decisions here are load-bearing for the whole editor: **the collection owns its elements**,
and **an element's identifier is generated, not asked for**.

## State

See [`property_collection.hpp`](property_collection.hpp.md). The list and the change flag
are the owner's; the *elements in the list* are this adapter's.

## Element identity and the two-sided reference

Each element is reachable two ways: as a model object (the thing with fields the editor
writes into the configuration file) and as a property holder (the thing the grid shows).
The collection stores model objects; the grid hands back property holders; converting
between them is the adapter's constant chore.

```text
# each element answers both questions
element.property_holder()   # the grid's view of it
holder.owner()              # back to the model object
```

**Notes** — This double-ended reference exists because the grid and the model live in
different modules and neither may hold the other's type by value. A rebuild in one
language stores the element once and drops the conversion entirely; what must survive is
that **the grid's row identity and the model object's identity are the same identity** —
if they diverge, selecting a row edits the wrong keyframe.

## `insert`

**Contract** — puts an existing element at a position in the owner's list, marks the
owner changed, and takes over responsibility for it. The position may equal the list
length (append). The incoming value must actually be an element of this collection's type;
it is not, in the original, a checked runtime condition in release builds.

```text
FUNCTION insert(holder : PropertyHolder, position : int)
  mark_owner_changed()
  REQUIRE position <= length(elements)
  value = holder.owner()
  REQUIRE value IS an Element
  elements.insert_at(position, value)
```

## `erase` and `destroy`

**Contract** — `erase` removes the element at a position and releases it. `destroy`
releases an element the grid is holding without removing it from any list — the grid calls
it for an element it created but never inserted (an add that was cancelled).

```text
FUNCTION erase(position : int)
  mark_owner_changed()
  REQUIRE position < length(elements)
  value = elements[position]
  elements.remove_at(position)
  release(value)

FUNCTION destroy(holder : PropertyHolder)
  release(holder.owner())
```

**Notes** — Releasing inside `erase` is the decision. The alternative — the grid releases
what it removed — would put the model's lifetime in the hands of a user-interface widget
across a module boundary, which is where editors leak or double-free. **The list owns its
elements; removal is destruction.** Clearing the whole collection does *not* release them
here, which is an asymmetry worth noticing: `clear` empties the list and leaves the
elements to the owner's own teardown.

## `index`, `item`, `size`, `clear`

**Contract** — `index` finds the position of an element by its property-holder identity
and answers "not present" as a negative position. `item` returns the property holder at a
position. `size` is the element count. `clear` empties the list and marks the owner
changed.

## `unique_id` and `generate_unique_id`

**Contract** — `unique_id` answers whether no element already carries a given identifier.
`generate_unique_id` returns the first identifier of the form *prefix* followed by a
decimal counter that no element carries. Neither blocks; `generate_unique_id` always
terminates because the counter is unbounded.

```text
FUNCTION generate_unique_id(prefix : text) -> text
  i = 0
  WHILE true
    candidate = prefix + decimal(i)
    IF unique_id(candidate)
      RETURN candidate
    i = i + 1
```

**Notes** — Starting at zero and scanning upward, rather than counting elements, is what
makes the identifier stable across deletions: delete `sun_unique_id_3` and the next new
sun reclaims that name instead of colliding with `sun_unique_id_4`. The comparison is
exact text, so identifiers are case-sensitive throughout the weather model.

Keyframes do not use this rule — their identifiers are times of day, and
[`editor_environment_weathers_weather.cpp`](editor_environment_weathers_weather.cpp.md)
overrides it with a rule that keeps them parseable as clock times.

## `display_name` and `create`

**Contract** — the two operations the adapter cannot implement generically. `display_name`
writes an element's grid label into a caller-supplied text buffer. `create` builds a new
element, gives it a generated identifier, registers it with this collection, and returns
its property holder.

**Notes** — They are declared here and defined once per element type, beside that type's
owner. That placement is not arbitrary: `create` must know the element's constructor
arguments (which manager it belongs to, what identifier prefix its kind uses), and only
the owner's file knows those.
