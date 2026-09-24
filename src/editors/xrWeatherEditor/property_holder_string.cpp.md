# src/editors/xrWeatherEditor/property_holder_string.cpp

> Registers text properties, and picks which of three editing affordances — free text, a chooser, or a file browser — the user gets.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_string.hpp`](property_string.hpp.md) · [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_string_values_value.hpp`](property_string_values_value.hpp.md) · [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md) · [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md) · [`property_string_values_value_shared_str_getter.hpp`](property_string_values_value_shared_str_getter.hpp.md) · [`property_file_name_value.hpp`](property_file_name_value.hpp.md) · [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md) · [`property_editor_file_name.hpp`](property_editor_file_name.hpp.md) · [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) · [`property_converter_string_values.hpp`](property_converter_string_values.hpp.md) · [`property_converter_tree_values.hpp`](property_converter_tree_values.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: copies text across the runtime boundary in both directions and binds to the engine's interned-text store

## Purpose

Eight registration overloads for text properties. Text carries the references in a weather document — texture names, sound set names, the name of another weather set — so almost every text property here is really a reference to an authored asset, and the editing affordance matters more than the type.

## State

Stateless.

## `add_property` (text)

**Contract** — As in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md). The binding flavour here is not "callbacks versus field alias" but "callbacks versus **interned-text field**": reference-bound text aliases a slot in the engine's shared-text store, and writing it goes through the engine so the store's accounting stays correct. Text is copied in both directions at every crossing.

| affordance | selected by | value editor | converter |
|---|---|---|---|
| free text | no list, no file mask | none | none |
| file browser | a file mask is given | file chooser | none if typing allowed, else text-locking |
| chooser, flat | a list, combo-box requested | none | list-of-values, typing allowed or locked |
| chooser, tree | a list, tree-view requested | tree chooser | none if typing allowed, else text-locking |

```text
FUNCTION choose_presentation(has_file_mask, list_shape, may_type)
  IF has_file_mask THEN
    editor    = file_chooser
    converter = may_type ? none : text_locking
  ELSE IF list_shape == tree THEN
    editor    = tree_chooser
    converter = may_type ? none : text_locking
  ELSE                                        # flat list
    editor    = none
    converter = may_type ? list_of_values : list_of_values_locked
  RETURN (editor, converter)
```

**Notes** — "May the user type?" is a real authoring decision and not a convenience. A text property that names an asset must usually be one of the assets that exist, and letting the artist type a name that does not resolve produces a weather set that loads with a missing texture and no diagnostic. Where typing is forbidden, the property is given a converter whose sole job is to refuse text-to-value conversion — the grid then has no path to accept a typed string and the chooser becomes the only way in. That is the whole content of [`property_converter_tree_values.cpp`](property_converter_tree_values.cpp.md) and of the locked variant in [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md).

Flat versus tree is a scale decision, not a semantic one. A flat list drops into the grid row itself and is right for a handful of choices; a tree opens a modal window and is right for asset paths, where the list is long and hierarchical. The values are identical in both cases — the same flat sequence of strings, which the tree chooser splits on path separators itself.

The four list-bearing overloads come in snapshot and live flavours for the same reason as the whole-number ones: some lists are only known while the editor runs.
