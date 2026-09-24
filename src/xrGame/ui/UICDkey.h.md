# src/xrGame/ui/UICDkey.h

> Declares the masked product-key field, the player-name field, and the four per-machine
> settings operations behind them.

**Needs** — [`UICDkey.cpp`](UICDkey.cpp.md) · [`../../xrUICore/EditBox/UIEditBox.h`](../../xrUICore/EditBox/UIEditBox.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`Level_GameSpy_Funcs.cpp`](../Level_GameSpy_Funcs.cpp.md) · [`Level_start.cpp`](../Level_start.cpp.md) · [`MainMenu.cpp`](../MainMenu.cpp.md) · [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UICDkey.cpp`](UICDkey.cpp.md) · [`UIMapList.cpp`](UIMapList.cpp.md) · [`UIOptConCom.cpp`](UIOptConCom.cpp.md)
**Tier floor** — T1: reads and writes a per-machine settings store outside the game's data

## Purpose

Declares the surface implemented in [`UICDkey.cpp`](UICDkey.cpp.md).

## `CUICDkey`

A text field that holds the multiplayer product key. It is simultaneously an edit box and a
*settings control* — it implements the toolkit's four-step settings protocol (read, back up,
commit, undo; see chapter 15) against the per-machine store rather than against a console
variable, which is the one place that protocol is used for something that is not a console
variable.

It draws itself entirely by hand, because the key must be shown masked.

## `CUIMPPlayerName`

A text field that holds the player's multiplayer name. Writes to the per-machine store on
losing focus and does nothing else.

## The four free functions

`GetCDKey_FromRegistry`, `WriteCDKey_ToRegistry`, `GetPlayerName_FromRegistry`,
`WritePlayerName_ToRegistry` — the per-machine store operations, declared here because the
menus that need them do not otherwise know about this file.
