# src/xrGame/PhraseDialogManager.h

> Declares the conversation-participant mixin implemented in [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md).

**Needs** — [`PhraseDialogDefs.h`](PhraseDialogDefs.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`AI_PhraseDialogManager.h`](AI_PhraseDialogManager.h.md) · [`Actor.h`](Actor.h.md) · [`PhraseDialog.cpp`](PhraseDialog.cpp.md) · [`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) · [`UITalkWnd.cpp`](ui/UITalkWnd.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the surface a class must acquire to take part in a phrase dialog — the actor and
every talking creature inherit it. Substance is in
[`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md).

Exported units:

- `CPhraseDialogManager` — the participant mixin. Overridable: `InitDialog`, `AddDialog`,
  `ReceivePhrase` (the "partner spoke" hook, meaningful only in subclasses), `SayPhrase`,
  `UpdateAvailableDialogs` (rebuild the offer list for a partner) and `AddAvailableDialog`
  (consider one dialog id).
- `AvailableDialogs`, `GetDialogByID`, `HaveAvailableDialog` — read access to the offer
  list, which the talk screen presents in its stored order.
