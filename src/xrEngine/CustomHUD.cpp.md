# src/xrEngine/CustomHUD.cpp

> Defines the heads-up display's default feature set.

**Needs** — [`CustomHUD.h`](CustomHUD.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one initialized value.

## Purpose

Holds the module-wide HUD flag word and its shipped default. It is a separate file only because the interface header must not define storage; a rebuild puts the default next to the declaration.

## `hud_flags`

**Contract** — The default enables the render-target variants of crosshair, weapon and draw for both deferred renderer generations, plus the dynamic crosshair; the three legacy forward-renderer toggles start off. Every bit is subsequently overwritten from the user's saved settings if one exists. See [`CustomHUD.h`](CustomHUD.h.md) for what the bits mean.
