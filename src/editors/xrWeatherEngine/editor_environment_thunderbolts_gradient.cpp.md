# src/editors/xrWeatherEngine/editor_environment_thunderbolts_gradient.cpp

> One glow on a thunderbolt, addressed by a key prefix so the same record serves both ends of the bolt.

**Needs** — [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md)
**Tier floor** — T1: a grid edit re-creates a material binding on the device.

## Purpose

Two glows are drawn per thunderbolt, at its centre and at its top, with identical fields.
Rather than two record types, there is one that takes the key prefix as an argument.

## State

See [`editor_environment_thunderbolts_gradient.hpp`](editor_environment_thunderbolts_gradient.hpp.md).

## The prefixed key group

```text
RECORD GradientKeys                     # four keys inside a thunderbolt's section
  <prefix>_shader  : text
  <prefix>_texture : text
  <prefix>_opacity : real
  <prefix>_radius  : vec2      # minimum and maximum
```

where *prefix* is `gradient_top` or `gradient_center`.

**Invariants** — all four keys are required; there are no defaults, so a thunderbolt
section missing one fails to load.

**Notes** — **Prefixing keys instead of nesting sections is the configuration format's
only way to express a sub-record**, and this is the module's cleanest use of it: the prefix
is a parameter, not a constant, so adding a third glow would need no new code. A rebuild in
a format with real nesting drops the prefix and keeps the record.

## `load` and `save`

```text
FUNCTION load(config, section, prefix)
  shader  = config.text(section, prefix + "_shader")
  texture = config.text(section, prefix + "_texture")
  opacity = config.real(section, prefix + "_opacity")
  radius  = config.vec2(section, prefix + "_radius")
  flare.create_material(shader, texture)

FUNCTION save(config, section, prefix)
  write the same four keys under the same prefix
```

**Contract** — loading binds the material on the device immediately, so the glow is drawable
the moment it is read.

## `fill`

```text
FUNCTION fill(environment, name, description, parent_holder)
  property_holder = editor.create_property_holder(name)
  parent_holder.add_property(name, group "gradient", description, property_holder)
  add "opacity"        (real, bound to the field)
  add "minimum radius" (real, bound to radius.min)
  add "maximum _radius" (real, bound to radius.max)
  add "shader"  (bound to accessors, options = environment.shader_ids, tree picker)
  add "texture" (bound to the field, file browser, .dds, extension dropped)
```

**Contract** — creates a nested page and attaches it to the parent, so a thunderbolt shows
`center` and `top` as expandable rows.

**Notes** — The maximum-radius row's label carries a stray underscore — `maximum _radius` —
which is visible to authors. Cosmetic, and a reminder that these labels are the model's
only documentation.

Note the asymmetry between the two resource rows: **the material row binds through an
accessor and re-creates the device material; the texture row binds to the field and does
not.** So changing a glow's material takes effect immediately and changing its texture does
not, until something else rebinds. That is a defect, not a design — the setter that would
fix it exists in this same file and the texture row simply does not use it.

## The two material setters

```text
FUNCTION set_shader(value : text)
  shader = value
  flare.create_material(shader, texture)

FUNCTION set_texture(value : text)
  texture = value
  flare.create_material(shader, texture)
```

**Notes** — Both re-create the material from the pair, because a material binding is the
two names together; changing either requires rebuilding it. Neither returns early on an
unchanged value.
