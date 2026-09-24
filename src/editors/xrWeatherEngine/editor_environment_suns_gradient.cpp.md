# src/editors/xrWeatherEngine/editor_environment_suns_gradient.cpp

> The soft halo around a sun — in code nothing currently reaches, with one abandoned experiment left in it.

**Needs** — [`editor_environment_suns_gradient.hpp`](editor_environment_suns_gradient.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_suns_gradient.hpp`](editor_environment_suns_gradient.hpp.md)
**Tier floor** — T3: five values and two pickers.

## Purpose

Unreachable, and worth keeping for its defaults and for one instructive dead end.

## State

See [`editor_environment_suns_gradient.hpp`](editor_environment_suns_gradient.hpp.md).

## The gradient record on disk

```text
RECORD GradientInASunSection          # keys inside a sun's own section
  gradient          : bool    # default true
  gradient_opacity  : real    # default 0.7
  gradient_radius   : real    # default 0.9
  gradient_shader   : text    # default "effects/flare"
  gradient_texture  : text    # default "fx/fx_gradient.tga"
```

**Notes** — The shipping defaults again act as the specification. The halo shares the
flare's material pass by default, which is why a sun that disables its flare series still
looks right — the material is not the series' property.

## `fill` and the abandoned experiment

```text
FUNCTION fill(manager, holder, collection)
  holder.add_property("use", group "gradient", bound to use_getter/use_setter,
                      read-write, NOTIFY PARENT ON CHANGE, no password, do not refresh grid)
  holder.add_property("opacity", group "gradient", bound to opacity)
  holder.add_property("radius",  group "gradient", bound to radius)
  holder.add_property("shader",  group "gradient", bound to shader,
                      options = manager.environment.shader_ids, tree picker, no free text)
  holder.add_property("texture", group "gradient", file browser, .dds, extension dropped)

FUNCTION set_use(value : bool)
  IF use == value THEN RETURN
  use = value
  # the original then attempts, and abandons, rebuilding the page from scratch
```

**Contract** — five rows in a group named `gradient`, added to the caller's holder.

**Notes** — The `use` row is the **only** row anywhere in this module that asks the grid to
notify the parent on change, and its setter's abandoned body says why: the intent was that
switching the halo off would *remove the other four rows*, so the author would not be
editing fields that do nothing. The mechanism — clear the holder, repopulate it — is
present as a commented pair of calls and does not run.

That is worth recording as a design question a rebuild must answer: **does a switch hide
the fields it disables, or leave them visible and inert?** The original wanted the first,
shipped the second, and left both the notification flag and the dead rebuild behind as
evidence. The second is the easier and arguably better answer — hiding rows makes the page
jump under the author's cursor — but the flag should then be dropped.

## `save` — declared and never written

No definition exists. See
[`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md).
