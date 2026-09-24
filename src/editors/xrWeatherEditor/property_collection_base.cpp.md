# src/editors/xrWeatherEditor/property_collection_base.cpp

> An editable list of engine objects, presented to the grid as an ordinary indexed collection — add, remove, reorder — with every operation forwarded straight to the engine.

**Needs** — [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_collection_enumerator.hpp`](property_collection_enumerator.hpp.md)
**Used by** — [`property_collection.cpp`](property_collection.cpp.md) · [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`property_collection_getter.cpp`](property_collection_getter.cpp.md)
**Tier floor** — T1: it narrows abstract engine-side holders to their concrete implementation on every access, which is only sound because there is exactly one implementation.

## Purpose

A large part of the weather model is lists: a cycle's keyframes, a sun's flares, an ambient's sound and effect identifiers, the thunderbolt collections. The grid offers a standard list editor for such a row — a dialog with add, remove, and up/down buttons — provided the value behaves like an indexed collection.

This class makes the engine's [collection interface](../../Include/editor/property_holder_base.hpp.md) behave like one. Every operation is a forward; the substance is **which forward, and what it means for ownership**.

## State

```text
RECORD CollectionProperty
  # no elements stored: the engine's collection is the only copy
  collection() -> EngineCollection        # supplied by the subclass
```

**Invariants** — this object holds *nothing*. Count, contents and order are read from the engine's collection on every call. The list editor's dialog therefore cannot show a stale list, and there is no commit step.

**Notes** — the class is abstract solely so that the collection can be reached two ways: [held directly](property_collection.cpp.md), or [fetched through a callable](property_collection_getter.cpp.md) each time. Everything else is here.

## The element mapping

**Contract** — the engine's collection holds *holders*; the grid must see *containers* (the objects its property bag binds to). Every read narrows a holder to the concrete [`property_holder`](property_holder.hpp.md) and returns its container; every write takes a container and passes back its holder.

```text
FUNCTION element_at(position) -> Container
  RETURN as_concrete(collection().item(position)).container

FUNCTION holder_of(container) -> Holder
  RETURN container.holder
```

**Notes** — the narrowing is checked in debug and unchecked in release (see [`pch.hpp`](pch.hpp.md)). It is sound because the editor library is the only thing that ever creates a holder, through [`ide_impl.create_property_holder`](ide_impl.cpp.md). A rebuild that keeps the interface should carry the container on the interface instead, and delete the narrowing.

## The list operations

**Contract** — the standard indexed-collection surface, each forwarded:

```text
FUNCTION count()             -> collection().size()
FUNCTION index_of(container) -> collection().index(holder_of(container))
FUNCTION contains(container) -> index_of(container) >= 0
FUNCTION insert(position, container) -> collection().insert(holder_of(container), position)
FUNCTION remove_at(position) -> collection().erase(position)
FUNCTION remove(container)   -> remove_at(index_of(container))
FUNCTION clear()             -> collection().clear()
FUNCTION replace(position, container)
  remove_at(position) ; insert(position, container)
```

**Invariants** — `remove_at` *erases* without destroying. That is the collection interface's one non-obvious rule, stated in [`property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md): removal and destruction are separate so that a reorder — erase then insert — cannot destroy the element in between. Everything here depends on it; `replace` is literally that pair.

**Notes** — `append` returns the count *before* the insert minus one, which is off by two from the position it actually inserted at (it inserts at the end, so the correct answer is the old count). The return value is ignored by the grid's list editor, which is why nobody noticed. A rebuild returns the insertion position.

## `create`

**Contract** — asks the engine's collection to make a new element and returns its container. **Creating does not insert**: the new element exists, unowned by the list, until the grid's dialog inserts it where the author asked.

**Notes** — this is the "add" button's implementation, and the separation is what lets the dialog place a new element at the selection rather than always at the end.

## Iteration

**Contract** — hands back [an enumerator](property_collection_enumerator.cpp.md) over the engine's collection. Copying out to an array walks the collection and writes each element's container.

**Notes** — the copy-out walk starts at the source position rather than at zero and writes to the same index, so a copy requested into the middle of a target array writes the *wrong elements* into the right slots. It is unexercised — the grid copies the whole collection to position zero, where the bug is invisible. A rebuild indexes the source from zero and the target from the offset.

## `get` / `set` as a grid value

**Contract** — reading the property yields the collection object itself, so the grid's list editor gets something to edit. Writing is a no-op: a collection row is never assigned wholesale, only mutated through the dialog.

## Notes

The declared-but-unused parts of the standard collection surface — a synchronization root, a "is this thread-safe" answer of no, fixed-size and read-only answers of no — exist because the grid's list editor demands the full interface. They carry no decision. What they do say, accurately, is that **the collection is not thread-safe and must only be touched from the thread that owns the editor's loop**, which is the same thread that runs the engine's frames.
