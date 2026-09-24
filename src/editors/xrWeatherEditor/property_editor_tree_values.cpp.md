# src/editors/xrWeatherEditor/property_editor_tree_values.cpp

> Opens a long list of admissible values as a browsable tree, and stores what the artist picked.

**Needs** — [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`window_tree_values.h`](window_tree_values.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure presentation; text crosses to the row through its own interface

## Purpose

The alternative to a drop-down for text rows whose admissible set is too long to scroll. The set is a flat sequence of names; the chooser window is what gives it a shape, by treating the separators inside each name as a hierarchy.

## State

```text
RECORD TreeChooser
  window : chooser window      # one instance, reused across every row and every open
```

Reused rather than built per open, so the window keeps its size and scroll position between uses — an artist comparing several textures opens the same list repeatedly, and a chooser that reset each time would be worse than a drop-down.

## `GetEditStyle`

**Contract** — Modal whenever there is a row context; otherwise the grid's default.

## `EditValue`

**Contract** — Asks the row for its admissible set and its current value, opens the chooser on both, and on confirmation writes the picked value back through the row's adapter. Blocks. Does nothing on cancel. Returns the value it was handed, unchanged.

```text
FUNCTION EditValue(context, services)
  IF no context OR no services OR no window service THEN RETURN default

  adapter = context.subject.property(context.row)    # AS publishes_admissible_set
  window.load(adapter.values(), adapter.GetValue())  # list, and what to preselect
  IF window.show() == confirmed THEN
    adapter.SetValue(window.result)
  RETURN default
```

**Notes** — The set is fetched at the moment of opening, not cached by this editor, which is what lets a row whose admissible set is regenerated per query ([`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md)) show what exists *now* rather than what existed when the row was built.

The current value is passed alongside the list so the chooser can open with it selected and its branch expanded. Opening a several-hundred-entry tree collapsed at the root, with no indication of where the current value sits, makes "which texture is this" unanswerable without closing the dialog — which is precisely the question the chooser exists to answer.

The window is driven entirely through its own two-step protocol: load the list and the selection, then show. That keeps this file free of any knowledge of how the names become a tree; the splitting rule belongs to [`window_tree_values.h`](window_tree_values.h.md).
