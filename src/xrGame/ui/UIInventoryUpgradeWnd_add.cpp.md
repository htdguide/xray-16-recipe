# src/xrGame/ui/UIInventoryUpgradeWnd_add.cpp

> The upgrade bench's two loaders, split out of its implementation: the closed vocabulary of cell-state names, and the construction of every authored tree shape.

**Needs** — [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIInvUpgrade.h`](UIInvUpgrade.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md)
**Tier floor** — T3.

## Purpose

A second implementation file for one class, containing only its construction-time reading.
The split is **arbitrary** — it is a file-size decision, not a design one — and a rebuild
should merge it back into
[`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md). It is documented separately only
because the mirror must be complete; the decisions in it are described in that twin, under
*The view-state texture tables* and *Building the schemes*.

## `LoadCellsBacks` / `LoadCellStates` / `SelectCellState` / `SetCellState`

**Contract** — read the layout's state list into the two texture tables. Each entry gives a
state name, a cell background texture, a point texture and an item colour. Empty texture
strings are normalised to absent. An unrecognised state name is fatal.

**Invariants** — the state-name vocabulary is closed and frozen against the shipped
documents; two of the ten names differ from the corresponding code identifiers
(`highlight` for focused, `disabled_highlight` for disabled-focused).

**Notes** — the item colour is read and discarded. A node's icon tint is decided by its
verdict in code, so the authored colour has no effect. Dead data in every shipped document.

## `VerirfyCells`

**Contract** — reports whether all ten cell textures were supplied.

**Notes** — never called. A document missing a state leaves that state invisible rather than
being rejected. A rebuild should call it. (The name's spelling is the original's.)

## `LoadSchemes`

**Contract** — read one shared cell rectangle and one optional shared border rectangle,
scaling **widths only** by 0.8 on a widescreen display, then build every template as a
scheme of columns of cells, each cell a node and, where the document authors a point offset,
a marker attached to it.

**Invariants** — a node's identity within its scheme is its (column index, cell index) pair,
and that pair is what the upgrade registry is later asked to resolve into an upgrade for a
given item. **The document's iteration order is therefore load-bearing**: reordering columns
or cells in the layout re-binds every upgrade.
