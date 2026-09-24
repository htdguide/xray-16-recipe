# src/xrGame/ui/UIStatsIcon.cpp

> A scoreboard icon cell, resolving a short string into one of eight pre-resolved
> (material, sub-rectangle) pairs held in a process-wide table.

**Needs** — [`UIStatsIcon.h`](UIStatsIcon.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIStatsIcon.h`](UIStatsIcon.h.md)
**Tier floor** — T1: holds graphics-device material handles that must be released at a defined point

## Purpose

Every player row in the scoreboard carries two icon cells — a rank badge and a status marker — and a
round can have dozens of rows repainted every tenth of a second. Resolving an icon name to a
material and a sub-rectangle on every repaint would mean a registry lookup per cell per update, so
the eight possible icons are resolved **once** into a table shared by every cell in the process, and
a repaint is two field assignments.

## State

```text
RECORD IconEntry
  material  : MaterialHandle       # from the graphics device; must be released
  rect      : Rect                 # sub-rectangle of the atlas page

# process-wide, built on first cell construction, released explicitly
table : optional<map<IconKind, IconEntry[2]>>   # index 2 == team: 0 green, 1 blue
```

Invariants:

- The table is built at most once and is shared by every cell; the first cell constructed builds it.
- It holds graphics-device material handles, so it must be released explicitly — which is done from
  the player list's teardown, not from a cell's. A cell outliving the table would draw with a
  released material.
- Both team variants of the artefact and death entries are copies of the green one, so the
  team index is accepted and ignored for those two.

## The eight icons

**Contract** — Five rank badges, an artefact-carrier marker and a death marker. The ranks are
addressed by registered icon names built from a template: a colour word and a one-based two-digit
number — `ui_hud_status_green_01` … `_05` and the blue equivalents. The artefact marker is not a
registered icon at all: it is cut out of the **inventory icon atlas** at the grid cell the artefact
hunt game mode's configured artefact item occupies, so it is literally the item's inventory picture.
The death marker is a hand-placed 30×30 rectangle at (32, 202) in a named texture.

```text
FUNCTION InitTexInfo()
  IF table EXISTS THEN RETURN
  FOR rank IN 0 .. 4
    table[rank][green] <- registry lookup of "ui_hud_status_green_0" + (rank+1)
    table[rank][blue]  <- registry lookup of "ui_hud_status_blue_0"  + (rank+1)

  artefact <- the item section the artefact-hunt configuration names
  table[ARTEFACT][green].material <- the inventory icon atlas material
  table[ARTEFACT][green].rect     <- artefact's inventory grid cell, in atlas pixels
  table[ARTEFACT][blue]           <- table[ARTEFACT][green]

  table[DEATH][green].material <- material over the kill-marker texture
  table[DEATH][blue]           <- table[DEATH][green]
  table[DEATH][green].rect     <- (32, 202) .. (62, 232)
```

**Notes** — The death marker's rectangle is four literal numbers with no discoverable source: it is
the position of one sprite inside a shared multiplayer icon sheet, and nothing in the repository
records why it is there rather than being a registered icon like the ranks. A rebuild must copy the
numbers or re-register the sprite.

The rank table is declared with six slots and filled with five; the sixth is never written and never
read. The inventory grid cell arithmetic multiplies the item's grid coordinates by the atlas's cell
size — the same conversion the inventory screens use, shared through the inventory utility module.

The blue variant of the artefact and death entries is a **copy of the green one including the
material handle**, so releasing the table releases the same handle twice. That is a real defect in
the shipped code, survivable only because the release path runs once at scoreboard teardown; a
rebuild should hold one handle and two rectangles.

## `CUIStatsIcon`

**Contract** — A picture cell that stretches its texture to the cell rectangle and ensures the
shared table exists. Construction is cheap after the first.

## `SetValue`

**Contract** — Turns the short string a scoreboard field produced into a material and a rectangle.
An empty string hides the cell; anything else shows it. The string is classified by content, not by
an enumeration:

```text
FUNCTION SetValue(name)
  IF name IS empty THEN visible <- false; RETURN
  visible <- true

  IF name CONTAINS "status" THEN                  # a rank badge name
    team <- 0 IF name CONTAINS "green" ELSE 1
    rank <- (the digits following the first '0' in the name) - 1
    apply table[rank][team]
  ELSE IF name == "death"    THEN apply table[DEATH][0]
  ELSE IF name == "artefact" THEN apply table[ARTEFACT][0]
  ELSE                            bind the texture registered under that name
```

**Notes** — Classifying by substring means the *producer* of the string — the player-row field
formatter in [`UIStatsPlayerInfo`](UIStatsPlayerInfo.cpp.md) — and this consumer are coupled through
the shape of a name rather than through a type. The rank is recovered by parsing the digits back out
of the name it was just formatted into. A rebuild is free to pass a rank and a team instead; nothing
outside these two files observes the string.

The final fallback — bind whatever is registered under that name — is what lets a layout put an
arbitrary registered icon in a scoreboard column without the engine knowing about it.

## `FreeTexInfo`

**Contract** — Releases every material handle in the table and drops it. Called from the player
list's teardown. Safe to call when the table was never built.
