# src/xrGame/Phrase.h

> Declares the phrase record, implemented in [`Phrase.cpp`](Phrase.cpp.md).

**Needs** — [`PhraseScript.h`](PhraseScript.h.md)
**Used by** — [`Phrase.cpp`](Phrase.cpp.md) · [`PhraseDialog.cpp`](PhraseDialog.cpp.md) · [`PhraseDialog.h`](PhraseDialog.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPhrase`, one vertex of a dialog graph. Substance is in
[`Phrase.cpp`](Phrase.cpp.md).

The shape the header fixes: every field is settable from outside, because phrases are built
two ways — parsed from the dialog XML, or created one at a time by a script that assembles a
dialog at run time. Both paths go through the same mutators, so neither is privileged.

Exported units:

- `CPhrase` — the record: identifier, text, script-text function name, cached script text,
  goodwill threshold, finalizer flag, and the script helper.
- `SetText` / `GetText` / `GetScriptText` — the text surface.
- `SetID` / `GetID` — identity within one dialog. `"0"` names the entry phrase by convention.
- `SetFinalizer` / `IsFinalizer` — whether saying this ends the conversation.
- `SetGoodwillLevel` / `GetGoodwillLevel` / `GoodwillLevel` — the standing threshold, exposed
  under two names that do the same thing.
- `IsDummy` — has no text from any source, so it must not be offered to a player.
- `GetScriptHelper` — hands out the mutable script hooks so the loader can fill them.

The dialog is declared a friend so it may reach the script-text fields directly; that is a
consequence of the text resolution living in the dialog rather than here, and a rebuild is
free to move the resolution into the phrase and drop the special access.
