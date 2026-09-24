# src/editors/xrWeatherEngine/editor_environment_suns_blend.cpp

> Three numbers controlling how a sun's flare fades as the sun enters and leaves the view — in code nothing currently reaches.

**Needs** — [`editor_environment_suns_blend.hpp`](editor_environment_suns_blend.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_suns_blend.hpp`](editor_environment_suns_blend.hpp.md)
**Tier floor** — T3: three values.

## Purpose

The smallest of the four unreachable sun sub-records, and the only one whose content is
entirely three numbers. It survives in the recipe because the numbers' **defaults** are
the specification of the flare fade, and are recorded nowhere else.

## State

See [`editor_environment_suns_blend.hpp`](editor_environment_suns_blend.hpp.md).

## The blend record on disk

```text
RECORD BlendInASunSection               # keys inside a sun's own section
  blend_down_time : real    # default 60
  blend_rise_time : real    # default 60
  blend_time      : real    # default 0.1
```

**Notes** — The two long times and the short one are different quantities despite the
shared prefix: the rise and fall are the *rate* at which the flare's visibility catches up
when the sun becomes occluded or clear, and the short one is the per-frame step. Nothing
in this file or its neighbours states units. Sixty and a tenth are the shipping values; a
rebuild that changes them changes how every unmodified sun's flare behaves.

## `load` and `fill`

```text
FUNCTION load(config, section)
  down_time = config.real_or(section, "blend_down_time", 60)
  rise_time = config.real_or(section, "blend_rise_time", 60)
  time      = config.real_or(section, "blend_time", 0.1)

FUNCTION fill(manager, holder, collection)
  holder.add_property("down time", group "blend", bound to down_time)
  holder.add_property("rise time", group "blend", bound to rise_time)
  holder.add_property("time",      group "blend", bound to time)
```

**Contract** — three unbounded real rows in a group named `blend`, added to whatever holder
the caller supplies rather than a holder of its own — so the blend's rows appear inline
among the sun's, grouped.

**Notes** — Adding rows to the caller's holder rather than creating a nested page is the
pattern the gradient and flare-series records use too: **a sub-record with a handful of
fields becomes a group, not a page.** Only a sub-record with a *list* — the flare series —
earns a page of its own. That is a good rule for a rebuild to keep.

## `save` — declared and never written

No definition exists. One of the three reasons the suns file is not saved; see
[`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md).
