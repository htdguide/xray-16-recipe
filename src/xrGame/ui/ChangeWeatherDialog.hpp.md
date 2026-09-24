# src/xrGame/ui/ChangeWeatherDialog.hpp

> A numbered button list with keyboard shortcuts, and the two vote dialogs built on it.

**Needs** — [`ChangeWeatherDialog.cpp`](ChangeWeatherDialog.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Used by** — [`ChangeWeatherDialog.cpp`](ChangeWeatherDialog.cpp.md) · [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md)
**Tier floor** — T3: a button list and a console command

## Purpose

Declares the surface implemented in [`ChangeWeatherDialog.cpp`](ChangeWeatherDialog.cpp.md).

## `ButtonListDialog`

The shared shape: a background, a header, a cancel button, and **a variable number of
(button, label) pairs** created at initialisation. Its one behaviour beyond routing clicks is
that the number keys select buttons by position.

An implementor supplies only `OnButtonClick(index)`.

## `ChangeWeatherDialog`

Vote to change the weather. One button per weather set the game offers, each carrying the
set's name and its start time.

## `ChangeGameTypeDialog`

Vote to change the game type. A **fixed four** buttons, whose identifiers come from the layout
document rather than from any table the engine owns — a gap the source flags.
