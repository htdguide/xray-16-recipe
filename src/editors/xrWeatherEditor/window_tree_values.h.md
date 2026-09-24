# src/editors/xrWeatherEditor/window_tree_values.h

> Declares the picker dialog for a long, path-shaped list of names: a tree, a read-only echo of the choice, and accept or cancel.

**Needs** — [`window_tree_values.cpp`](window_tree_values.cpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — [`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md) · [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) · [`window_tree_values.cpp`](window_tree_values.cpp.md)
**Tier floor** — T3: a modal dialog over a name list.

## Purpose

Declares the surface implemented in [`window_tree_values.cpp`](window_tree_values.cpp.md).

This is the dialog behind every grid row that picks from a list of *hundreds* of
hierarchical names — material passes, particle systems, light-animation curves. The
short-list rows use a combo box instead; the choice between the two is made per row on the
engine side, and the rule is stated in
[`editor_environment_effects_effect.cpp`](../xrWeatherEngine/editor_environment_effects_effect.cpp.md).

## State

```text
RECORD TreePicker
  tree      : TreeView      # the names, split into folders
  echo      : TextBox       # read-only; shows the current choice
  ok        : Button
  cancel    : Button
  result    : text          # the chosen name, or empty for a folder
  images    : three icons   # closed folder, open folder, leaf
```

**Invariants** — `result` is empty whenever the selection is a folder rather than a name.

## Layout

The tree fills the dialog; the echo and the two buttons sit below it. The dialog is modal,
centred on its parent, has no icon and no taskbar entry, and is titled
`Select from the list...`.

## Exported units

- **`values`** — load the list and pre-select the current choice.
- **`result`** — the chosen name.
