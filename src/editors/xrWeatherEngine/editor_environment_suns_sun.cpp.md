# src/editors/xrWeatherEngine/editor_environment_suns_sun.cpp

> One sun record — six of its fields, which is why the suns file is never written back.

**Needs** — [`editor_environment_suns_sun.hpp`](editor_environment_suns_sun.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_suns_sun.hpp`](editor_environment_suns_sun.hpp.md)
**Tier floor** — T2: it is the engine's lens-flare record, extended.

## Purpose

A sun record describes what the sun looks like: the disc, and the flares that streak across
the view when it is in frame. This editor authors **the disc only**.

## State

See [`editor_environment_suns_sun.hpp`](editor_environment_suns_sun.hpp.md).

## The part of the sun record this editor reads

```text
RECORD SunSection                       # one configuration section per sun
  section name       : text
  sun                : bool   # default true   — draw the disc
  sun_ignore_color   : bool   # default false  — ignore the keyframe's sun colour
  sun_radius         : real   # default 0.15
  sun_shader         : text   # default "effects/sun"
  sun_texture        : text   # default "fx/fx_sun.tga"
  # ... and everything else the engine's lens-flare loader reads: the gradient,
  #     the blend timings, the flare series. Not read here. Not written here.
```

**Contract** — every field has a default, so a sparse section loads. Reading and writing are
symmetric *for these six*, which is exactly the problem: a symmetric write of a partial
read is a lossy rewrite.

**Notes** — **This is the concrete reason the suns file is excluded from save.** The
defaults are the engine's own, reproduced here — a radius of 0.15 of the view, the sun
material and its texture. Reproducing defaults in a second place is its own hazard: change
them in the engine and the editor silently disagrees.

The texture default carries an extension while the row that edits it strips extensions, so
a sun left at its default and then edited loses the `.tga`. In practice the texture system
resolves both.

## `fill` — the sun's six rows

```text
GROUP common : id (text, filtered through the manager's naming rule)
GROUP sun    : use (checkbox)
               ignore color (checkbox)
               radius (real)
               shader (chosen from the model's material-pass names, tree picker)
               texture (file browser, .dds, extension dropped)
```

**Notes** — The shader row browses the material-pass list read out of the shader library by
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) — a few hundred
names with folder structure, hence the tree picker.

Nothing in this page reaches the flare series, the gradient or the blend timings, although
three files in this module were written to edit them:
[`suns_flares`](editor_environment_suns_flares.cpp.md),
[`suns_flare`](editor_environment_suns_flare.cpp.md),
[`suns_gradient`](editor_environment_suns_gradient.cpp.md) and
[`suns_blend`](editor_environment_suns_blend.cpp.md). None is reachable from here. A
rebuild that wires them in completes the record and unlocks the save.
