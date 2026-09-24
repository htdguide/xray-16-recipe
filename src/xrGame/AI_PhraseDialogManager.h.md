# src/xrGame/AI_PhraseDialogManager.h

> Declares the non-player conversation mix-in implemented in [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md).

**Needs** — [`PhraseDialogManager.h`](PhraseDialogManager.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai_trader.h`](ai/trader/ai_trader.h.md) · [`script_game_object2.cpp`](script_game_object2.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the behaviour every talking non-player character inherits: it extends the shared
dialogue-manager role with the ability to *choose* a reply. Substance is in
[`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md).

Exported units:

- `CAI_PhraseDialogManager` — the mix-in. Holds the current and default opening dialogue.
- `ReceivePhrase` — answer, then run the shared receive path.
- `AnswerPhrase` — pick the reply matching the speaker's attitude and say it.
- `UpdateAvailableDialogs` — seed the offer set with the opening and greeting dialogues.
- `SetStartDialog` / `GetStartDialog` / `SetDefaultStartDialog` / `RestoreDefaultStartDialog` —
  the scriptable opening-dialogue override and its restore.

## Notes

A pending-dialogue list is declared and never used anywhere in the codebase. A rebuild
should drop it; the queued-answer design it hints at was never implemented.
