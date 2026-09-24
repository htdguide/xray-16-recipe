# src/editors/xrWeatherEditor/property_collection_editor.cpp

> The add/remove/reorder dialog for a collection row: what a new element is, what each element is called, and the one repaint that keeps the rendered view honest while the dialog is open.

**Needs** — [`property_collection_editor.hpp`](property_collection_editor.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`ide_impl.hpp`](ide_impl.hpp.md) · [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_collection_editor.hpp`](property_collection_editor.hpp.md)
**Tier floor** — T1: it fills a fixed-size caller-supplied buffer with a display name, because a string cannot cross the module boundary by value.

## Purpose

The grid's stock list editor knows how to arrange a dialog with an element list, an add button, a remove button and reorder arrows. It does not know what an element of *this* list is, what to call one, or that something outside the dialog needs repainting. This supplies those four answers.

## State

```text
RECORD CollectionEditor
  dialog : optional<Dialog>      # the live dialog, if one is open
```

## `element_type`

**Contract** — answers "what type is an element" with the container type, which is what [`property_collection_base`](property_collection_base.cpp.md) hands out for every element.

## `create_element`

**Contract** — the add button. Walks from the dialog's context back to the collection binding behind the row, and asks *it* to create an element — so creation goes through the engine's collection, not through the dialog's own notion of constructing a value.

```text
FUNCTION create_element() -> Container
  container  = the object whose row is being edited
  descriptor = the row being edited
  binding    = container.property_for(descriptor.spec)
  RETURN as_collection(binding).create()
```

**Notes** — the walk is the same one [`PropertyGrid`](../xrSdkControls/Controls/PropertyGrid.cs.md) makes for its gestures, and it exists for the same reason: the dialog was handed a row, not a binding. What matters is the routing decision — **the engine creates the element, because only the engine knows what a keyframe or a flare is** — and that a newly created element is not yet in the list.

## `display_name_of`

**Contract** — the label shown for one element in the dialog's list. Asks the element's own collection for the name at that element's position; falls back to the holder's own display name when the element is not in a collection, or not in it yet.

```text
FUNCTION display_name_of(container) -> text
  holder = container.holder
  collection = holder.collection
  IF collection is absent THEN RETURN holder.display_name
  position = collection.index_of(holder)
  IF position < 0 THEN RETURN holder.display_name        # created but not yet inserted
  buffer = fixed 256-byte buffer
  collection.write_display_name(position, buffer)
  RETURN buffer as text
```

**Invariants** — the fallback is not decoration. An element created by the add button exists before it is inserted, and asking a collection for the name of an element it does not contain has no answer.

**Notes** — two things here are the boundary showing through and one is a real risk.

The name is written into a **caller-supplied fixed buffer** rather than returned, because a string cannot cross the module boundary by value — the same constraint that produces the text conversions in [`pch.hpp`](pch.hpp.md). The buffer is 256 bytes and **there is no indication of truncation**: a longer name is silently cut. Nothing in the weather model has a name near that long, which is why it is safe rather than why it is right. A rebuild returns a string.

The collection is asked for the element's *position* and then for the name *at* that position, two calls where one would do. That is the interface's shape, not a choice made here.

## The repaint hook

**Contract** — when the dialog is moved, the rendered three-dimensional view is invalidated.

**Notes** — this is the one line in the file that is about the editor rather than about lists, and it is worth understanding. **While a modal dialog is open, the engine's idle pump does not run** (see [`entry_point.cpp`](entry_point.cpp.md)), so the view is frozen and will not even redraw itself when the dialog uncovers part of it. Invalidating on every move is the minimum that keeps the uncovered area from showing stale pixels.

It is a symptom, not a fix — the view is still frozen, just not corrupt. A rebuild that inverts control, or that runs a modeless panel instead of a modal dialog, deletes the hook and the symptom together.

## Re-entrant editing

**Contract** — opening the editor while its own dialog is already visible creates a *second*, independent editor and opens that instead of reusing this one.

**Notes** — this is how a nested collection is edited: a keyframe list's element contains a flare list, and opening the inner one while the outer dialog is up must not disturb the outer. The stock editor holds one dialog per editor instance, so a second instance is the available answer.

The construction guard against already having a dialog is commented out in the source — it was presumably what this re-entrancy path was added to replace.
