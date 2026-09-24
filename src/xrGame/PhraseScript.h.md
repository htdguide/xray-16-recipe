# src/xrGame/PhraseScript.h

> Declares the per-phrase script and information gate implemented in [`PhraseScript.cpp`](PhraseScript.cpp.md).

**Needs** — [`InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md)
**Used by** — [`InfoPortion.cpp`](InfoPortion.cpp.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`Phrase.cpp`](Phrase.cpp.md) · [`Phrase.h`](Phrase.h.md) · [`PhraseDialog.cpp`](PhraseDialog.cpp.md) · [`PhraseDialog_script.cpp`](PhraseDialog_script.cpp.md) · [`PhraseScript.cpp`](PhraseScript.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the bundle embedded in every dialog and every phrase: the predicates that gate it
and the effects that fire when it is chosen. Substance is in
[`PhraseScript.cpp`](PhraseScript.cpp.md); the script-facing appenders are registered in
[`PhraseDialog_script.cpp`](PhraseDialog_script.cpp.md).

Exported units:

- `CDialogScriptHelper` — the bundle. Exported to scripts under the name `CPhraseScript`.
- `Load` — fill the bundle from one node of a dialog XML file.
- `Precondition` / `Action` — each in a one-speaker (dialog-level) and a two-speaker
  (phrase-level) form; the two forms differ in script signature and in effect ordering.
- `GetScriptText` — substitute a script-computed display text for the authored one.
- `Preconditions` / `Actions` — read access to the two name lists.
- `AddPrecondition`, `AddAction`, `AddHasInfo`, `AddDontHasInfo`, `AddGiveInfo`,
  `AddDisableInfo`, `SetScriptText` — the script-side builders.
- `CheckInfo` / `TransferInfo` — overridable information check and transfer, so a subclass
  can redirect them away from the actor.
