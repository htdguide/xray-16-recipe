# src/editors/xrWeatherEngine/editor_environment_thunderbolts_thunderbolt.cpp

> One thunderbolt: the mesh that is drawn, the sound that follows, the light curve that flashes, and the two glows at its ends.

**Needs** — [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md)
**Tier floor** — T2: it is the engine's thunderbolt description, extended.

## Purpose

A thunderbolt is four references and two nested sub-records. This file fixes its on-disk
shape and its authoring page, and it is the clearest example in the module of the base
loader calling *back* into the editor to build the pieces it needs.

## State

See [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md).

## The thunderbolt record on disk

```text
RECORD ThunderboltSection             # one configuration section per thunderbolt
  section name          : text
  color_anim            : text   # a light-animation curve name
  lightning_model       : text   # a mesh path, WITH its extension
  sound                 : text   # a sound path, extension dropped
  gradient_top_shader   : text   # ) the top glow, four keys with a shared prefix
  gradient_top_texture  : text   # )
  gradient_top_opacity  : real   # )
  gradient_top_radius   : vec2   # ) minimum and maximum
  gradient_center_*     : ...    # the same four keys with the prefix "gradient_center"
```

**Invariants** — the two gradients share one key shape distinguished only by prefix; see
[`editor_environment_thunderbolts_gradient.cpp`](editor_environment_thunderbolts_gradient.cpp.md).
Every key is required — there are no defaults.

**Notes** — **This layout is frozen.** Note the misspelling: the mesh key is
`lightning_model`, and the editor's own field and row label call it *lighting* model. The
file's spelling is the one that must be written.

## `load` and `save`

```text
FUNCTION load(config)
  base_description.load(config, id)     # calls create_top_gradient and create_center_gradient
  color_animator = config.text(id, "color_anim")
  lighting_model = config.text(id, "lightning_model")
  sound          = config.text(id, "sound")

FUNCTION save(config)
  top.save(config, id, prefix "gradient_top")
  center.save(config, id, prefix "gradient_center")
  config.write_text(id, "color_anim", color_animator)
  config.write_text(id, "lightning_model", lighting_model)
  config.write_text(id, "sound", sound)
```

**Invariants** — `save` dereferences both gradients unconditionally. A thunderbolt created
through the grid's add command has neither, because nothing calls the loader on it — so
saving a newly added thunderbolt fails. See the warning in
[`editor_environment_thunderbolts_manager.cpp`](editor_environment_thunderbolts_manager.cpp.md).

## The two gradient callbacks

```text
FUNCTION create_top_gradient(config, section)
  REQUIRE section == id
  top = new EditableGradient()
  top.load(config, section, prefix "gradient_top")
  base_description.top_gradient = top

FUNCTION create_center_gradient(config, section)
  REQUIRE section == id
  center = new EditableGradient()
  center.load(config, section, prefix "gradient_center")
  base_description.center_gradient = center
```

**Contract** — the base loader calls these while reading the section, expecting plain glow
records; it gets editable ones, and the base record is pointed at them.

**Notes** — This is the substitution pattern one level deeper than the environment's: the
loader asks for a sub-record and the editor supplies an editable one, so the same object is
both what the renderer draws and what the grid edits. **Every editable object in this
module is reachable both ways, and that is the module's organising idea.**

## `fill`

```text
FUNCTION fill(environment, collection)
  property_holder = editor.create_property_holder(id, collection, owner = self)
  add "id"             (text, filtered through the manager's thunderbolt naming rule)
  add "color animator" (chosen from the model's light-animation curve names, tree picker)
  add "lighting model" (file browser, ".dm", start folder = the mesh folder,
                        EXTENSION KEPT)
  add "sound"          (file browser, ".ogg", start folder = the sound folder,
                        extension dropped)
  center.fill(environment, "center", into this holder)
  top.fill(environment, "top", into this holder)
```

**Contract** — four rows in a group named `properties`, then two nested pages, one per
gradient.

**Notes** — **The mesh row is the module's only browsable-file row that keeps its
extension**, and the reason is in the file format: the mesh key is written with `.dm` and
the engine resolves it literally, while sounds and textures are resolved without one. A
rebuild must preserve the distinction per field, not per picker.

The two gradients are filled *into this holder*, so they appear as nested pages named
`center` and `top` rather than as inline groups — the exception to the "a small sub-record
becomes a group" rule noted in
[`editor_environment_suns_blend.cpp`](editor_environment_suns_blend.cpp.md), and the
reason is that there are two of them with identical field names, which would collide in one
group.
