# src/editors/xrWeatherEngine/editor_environment_thunderbolts_thunderbolt_id.cpp

> One member of a thunderbolt set: a name, constrained to the thunderbolts the model defines.

**Needs** — [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`ide.hpp`](ide.hpp.md)
**Used by** — [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

Identical in shape to
[`effect_id`](editor_environment_ambients_effect_id.cpp.md) and
[`sound_id`](editor_environment_ambients_sound_id.cpp.md), over a third source.

Three near-identical files is the cost of not parameterising the type over its option
source; a rebuild writes one.

## State

See [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md).

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(id, collection, owner = self)
  property_holder.add_property(
      "thunderbolt", group "properties",
      value bound to id,
      options pulled from manager.thunderbolts_ids,
      editor = combo box, free text not allowed)
```

**Contract** — one row: a combo box over every thunderbolt name the model defines, pulled
at paint time, with typing disallowed.

**Notes** — Unlike the collections list the keyframe picks from, this option list has **no
empty entry** — a set member must name a real strike, because the engine resolves every
member when the set loads and fails hard on an unknown name. An entry added through the
grid nevertheless starts empty, so a set with a freshly added member will not reload. Same
hazard as the other two reference types, and sharper here because the failure is hard
rather than silent.
