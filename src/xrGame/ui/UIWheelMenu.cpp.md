# src/xrGame/ui/UIWheelMenu.cpp

> An unfinished radial menu: it loads its layout document and then returns failure. Nothing uses it.

**Needs** — [`UIWheelMenu.h`](UIWheelMenu.h.md)
**Used by** — [`UIWheelMenu.h`](UIWheelMenu.h.md)
**Tier floor** — T4: contributes nothing

## Purpose

A placeholder for a radial quick-select menu. The build method loads the layout document
`wheel_menu.xml` — tolerating its absence — and then unconditionally reports failure, having built
nothing. Nothing constructs the type.

It is recorded because the mirror must be complete and because the document name reserves a slot in
the layout vocabulary: a rebuild adding a radial menu should use it rather than inventing a name.

## State

`Stateless.`

## `InitFromXml`

**Contract** — Loads `wheel_menu.xml` if present, discards it, and reports failure.

**Notes** — The load is not entirely pointless in the original: it forces the document to be read,
which surfaces a malformed file at a predictable moment. Nothing depends on that, and a rebuild
should either implement the menu or delete the file.
