# src/xrGame/ui/UILoadingScreenHardcoded.h

> Six compiled-in loading-screen layouts and two compiled-in texture descriptions, so that a loading screen exists before any UI data is known to work.

**Needs** — [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md)
**Used by** — [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md)
**Tier floor** — T4: it is data with two selector functions.

## Purpose

Pure data: layout documents and texture descriptions embedded in the executable, in the same
format the shipped data uses, plus two selectors that pick the right pair.

It exists because the loading screen is the **first** thing drawn and must not depend on the
UI data being present or correct. Carrying a document rather than constructing widgets means
the fallback goes through exactly the same reader as the real path, so the two cannot drift.

## Selection

```text
FUNCTION layout_for_current_game() -> document
  wide := the display is widescreen
  CASE mounted game OF
    first  : IF wide THEN first_game_wide  ELSE second_game_narrow   # note: not its own
    second : IF wide THEN second_game_wide ELSE second_game_narrow
    third  : IF wide THEN third_game_wide  ELSE third_game_narrow

FUNCTION textures_for_current_game() -> document
  CASE mounted game OF
    first, second : the shared older texture description
    third         : the newer texture description
```

**Notes** — the first game has no narrow layout of its own; it falls back to the second
game's. Whether that is an oversight or a deliberate reuse is not recoverable, and the two
games' loading art is similar enough that it looks intentional.

## What the documents encode

**Invariants** — all six are authored in the **1024×768 virtual canvas**, which is chapter
15's fixed canvas, and they are the clearest surviving statement of how the widescreen
variants differ from the narrow ones:

- the narrow layouts fill the canvas with the background image;
- the widescreen layouts **inset the image** — it is drawn about a hundred units in from each
  edge — and fill the two margins with dedicated side-panel textures, then re-centre every
  text element and the progress bar within the inset region.

That is chapter 15's "separate widescreen variants of the layout documents" mechanism shown
in full: the canvas is stretched non-uniformly, so a wide display gets a *different document*
whose art is narrower in canvas units, rather than a letterboxed version of the same one.

The texture descriptions are the icon-registry entries those documents reference: a page file
name and, within it, named sub-rectangles in that page's own pixels. Both the background and
the progress-bar fill are cut from **one page each**, and the progress-bar fill is taken from
a strip below the background image on the same page — 506×4 in one game, 268×37 in the others.
The background variant is chosen per language in the older description, which is why that one
declares three background pages.

One element carries the loading-stage override flag, which is how one game forces its stage
line on regardless of the player's setting. See
[`UILoadingScreen.cpp`](UILoadingScreen.cpp.md).

**Notes** — every geometric constant in these documents is authored art placement and none of
it is derivable; a rebuild copies the documents verbatim. The progress bar's easing factor is
declared here and then **bypassed** by the implementation, which sets the bar forcibly —
dead data that would become live if loading ever joined the ordinary frame loop.
