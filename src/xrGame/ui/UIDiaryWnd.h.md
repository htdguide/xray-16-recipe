# src/xrGame/ui/UIDiaryWnd.h

> Nothing: the whole file is a disabled declaration of a journal screen that no build compiles.

**Needs** — _(none: the file declares nothing)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: the file contributes no code at all.

## Purpose

The file's entire body is commented out. It preserves the shape of an abandoned *diary*
screen — a two-pane journal with a filter tab strip, a source list on the left, an article
description scroll view on the right, and a news feed — whose job was later split between the
PDA's own tabs and the encyclopedia article renderer.

Nothing includes it for a declaration, so a rebuild drops the file. It is recorded here only
so the mirror is complete and so a reader who finds the name in the source tree learns that it
is dead rather than missing.

**Notes** — the dead declaration is still evidence of one decision that survived elsewhere:
the journal distinguished *unread* from *read* sections with two different marker sprites
placed at authored positions, and re-sorted the source list whenever the filter changed rather
than hiding rows. Both behaviours reappear in the surviving PDA tabs.
