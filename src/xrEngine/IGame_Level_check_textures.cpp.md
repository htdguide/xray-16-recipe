# src/xrEngine/IGame_Level_check_textures.cpp

> Reports the loaded level's texture budget, and records the limits it was authored against.

**Needs** — [`IGame_Level.h`](IGame_Level.h.md) · [`Render.h`](Render.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a report over numbers the renderer supplies.

## Purpose

A separate file for one diagnostic, which is arbitrary — a rebuild folds it into the level. What is worth keeping is the two budgets it names, because they are the only place the content-authoring limits are written down.

## `check_textures`

**Contract** — Asks the renderer for its texture memory usage and count, split into ordinary textures and light maps, and logs both. Tests each against a limit; the ordinary-texture limit no longer reports at all and the light-map limit reports only in a debug build.

```text
FUNCTION check_textures()
  (base_bytes, base_count, lightmap_bytes, lightmap_count) = renderer.texture_usage()
  log both counts and both sizes in kilobytes
  # Authoring budgets, per level:
  #   base textures : at most 400, or 64 MB
  #   light maps    : at most 8,   or 32 MB
  # Exceeding the base budget means reducing the number of distinct textures
  # (preferred) or their resolution. Exceeding the light-map budget means
  # reducing lightmap pixel density or moving more lighting to vertices.
```

**Notes** — Both checks were once fatal and both have been defanged: the base-texture message is commented out entirely and the light-map one survives only in a debug build. The budgets were authoring-time constraints for a 2007 graphics card and are meaningless on modern hardware, which is why they no longer stop anything. They are recorded here because they are the only statement in the repository of how much texture a level was meant to use — useful to a rebuilder sizing a streaming budget, and useless as a runtime check.
