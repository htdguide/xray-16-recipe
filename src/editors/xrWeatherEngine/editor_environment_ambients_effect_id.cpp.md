# src/editors/xrWeatherEngine/editor_environment_ambients_effect_id.cpp

> One entry in an ambient's effect list: nothing but a name, constrained to the effects the model actually defines.

**Needs** — [`editor_environment_ambients_effect_id.hpp`](editor_environment_ambients_effect_id.hpp.md) · [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) · [`ide.hpp`](ide.hpp.md)
**Used by** — [`editor_environment_ambients_effect_id.hpp`](editor_environment_ambients_effect_id.hpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

The smallest editable object in the model, and the reason the editable-list adapter is
parameterised the way it is: a list of *references* must be add-able and remove-able like
any other list, so each reference needs an identity and a grid row of its own.

## State

See [`editor_environment_ambients_effect_id.hpp`](editor_environment_ambients_effect_id.hpp.md).

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(id, collection, owner = self)
  property_holder.add_property(
      "effect", group "properties",
      value bound to id,
      options pulled from effects_manager.effects_ids,
      editor = combo box, free text not allowed)
```

**Contract** — one row: a combo box over every effect name the model defines, with typing
disallowed. The option list is pulled at paint time, so an effect renamed elsewhere shows
its new name here immediately.

**Notes** — Disallowing free text is the whole point of this object existing rather than
the ambient storing plain strings: **a reference that cannot be mistyped cannot dangle.**
An entry created by the grid's add command starts with an empty name, which is not a valid
effect — the author must pick one before saving, and nothing enforces that.

Renaming the effect *record* does not rewrite the references pointing at it. The author
sees the old name in this row and the picker no longer offers it. That is the cost of
storing references as text, and a rebuild that stores identities instead removes it.
