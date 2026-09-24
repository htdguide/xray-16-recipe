# src/xrGame/ui/UITextBanner.h

> Declares an animated text banner drawn straight to the font layer — not a widget, and not built.

**Needs** — [`UITextBanner.cpp`](UITextBanner.cpp.md)
**Used by** — [`UITextBanner.cpp`](UITextBanner.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`UITextBanner.cpp`](UITextBanner.cpp.md). **This pair is
excluded from the build** — its entries in the module's file list are commented out — and nothing
in the repository references it. It is recorded here because the mirror must be complete, and
because the effect it describes is one a rebuild may want.

Exported units:

- `EffectParams` — one effect's tuning: period, cyclic or one-shot, on or off, stage, and two spare
  parameters.
- `CUITextBanner` — the banner: a font, a colour, an alignment, and a set of active effects.
  - `SetStyleParams(style)` — get a writable handle on one effect's tuning, or clear them all.
  - `Update()` — advance every enabled effect's clock.
  - `Out(x, y, format, …)` — apply the effects and draw one formatted line.
  - `ResetAnimation(style)` — restart one effect.
