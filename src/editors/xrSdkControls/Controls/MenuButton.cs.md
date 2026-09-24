# src/editors/xrSdkControls/Controls/MenuButton.cs

> A button that drops a menu below itself instead of firing an action, marked with a small downward triangle.

**Needs** — _(none beyond the host widget toolkit)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a click handler and a few polygon fills.

## Purpose

Toolbar affordances in the editor that open a choice rather than perform an action — pick a weather cycle, pick a level. It is one behaviour and one decoration, which is why it is twenty lines and a single file.

## State

```text
RECORD MenuButton
  menu : ContextMenu      # owned, created empty, populated by the caller
```

## `Menu`

**Contract** — exposes the owned menu for the caller to fill. Read-only: the button creates the menu and keeps it for its lifetime, so the caller never has to decide who releases it.

## Click and paint

**Contract** — a click drops the menu anchored at the button's bottom-left corner and then still raises the ordinary click notification, so a caller may react to the press as well. Paint draws the button normally, then a small filled triangle inset from the right edge, greyed when the button is disabled.

**Notes** — the source carries a commented-out variant that measured the screen and flipped the menu upwards when it would not fit below. It was abandoned; a rebuild on a toolkit whose menus reposition themselves gets that for free, and on one whose menus do not, the abandoned code is the correct behaviour to restore.
