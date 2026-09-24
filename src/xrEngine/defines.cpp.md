# src/xrEngine/defines.cpp

> The startup values of the global display mode and feature flags.

**Needs** — [`defines.h`](defines.h.md)
**Used by** — [`defines.h`](defines.h.md)
**Tier floor** — T3: three initializers

## Purpose

The definitions behind [`defines.h`](defines.h.md), and — the only thing worth recording —
the values the engine starts with before any configuration file or command line is read.

## State

```text
display_mode = {
  monitor        = 0            # the primary display
  window_style   = WINDOWED_BORDERLESS
  width          = 0            # 0: adopt the display's current mode
  height         = 0
  refresh_rate   = 0
  bits_per_pixel = 32
}

device_flags = { rsDrawStatic, rsDrawDynamic, rsDrawDetails, rsDrawParticles,
                 mtSound, mtNetwork }

texture_lod = 1                 # one mip level dropped from every texture
```

**Notes** — four decisions live in these defaults.

**Zero means "ask the platform".** A fresh installation with no settings file adopts the
display's current resolution and refresh rate rather than guessing one. Every consumer of
the mode record must therefore handle zero, not treat it as an error.

**Borderless windowed is the default**, not exclusive fullscreen — the engine prefers the
mode that always comes up over the one that is faster.

**Two of the four threading flags are on by default and two are off.** Sound and network
run off the main thread; physics and particles do not. The source notes that the shipping
build "always has the `mt_*` flags enabled", which is aspirational rather than descriptive:
what is actually enabled by default is these two. Physics and particles were left off, and
the source does not say why — the honest reading is that their worker paths were not trusted.

**A texture detail level of 1 means one mip is dropped from every texture at load.** It is
not "full quality"; full quality is zero. The default sacrifices texture sharpness for
memory out of the box, which mattered on the 32-bit builds this default was chosen for and
matters much less now. A rebuild targeting current hardware should default it to zero, and
should know that doing so changes the memory figures every other default was balanced
against.
