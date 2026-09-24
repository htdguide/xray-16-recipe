# src/xrGame/PhraseDialog_script.cpp

> Exports the dialog, phrase and phrase-script records to the script layer, so that a mod can build a conversation at run time instead of authoring it in data.

**Needs** — [`PhraseDialog.h`](PhraseDialog.h.md) · [`PhraseScript.h`](PhraseScript.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration table plus trivial appenders

## Purpose

The dialog system is normally fed from authored XML. This file opens the same records to
scripts, which is how the shipped games generate dynamic conversations (task offers,
trader haggling) whose phrase set is not known until run time. It also carries the
one-line appenders for the phrase-script helper's string lists, which live here rather
than beside the rest of that class only because they exist solely for the script surface.

## State

`Stateless.`

## exported surface

**Contract** — three types reach the script layer.

```text
Phrase
  GetPhraseScript() -> PhraseScript       # the phrase's script helper, for the calls below

PhraseDialog
  AddPhrase(text, phrase_id, prev_phrase_id, goodwill_level) -> Phrase
        # appends a phrase node under an existing one; goodwill_level gates it by the
        # partner's attitude. Returns the new phrase so the script can attach behaviour.

PhraseScript
  AddPrecondition(function_name)     # a script predicate gating the phrase
  AddAction(function_name)           # a script call fired when the phrase is said
  AddHasInfo(info_id)                # require the actor to hold this information portion
  AddDontHasInfo(info_id)            # require the actor not to hold it
  AddGiveInfo(info_id)               # grant it when the phrase is said
  AddDisableInfo(info_id)            # revoke it when the phrase is said
  SetScriptText(function_name)       # replaces the phrase's display text at show time
```

**Notes** — each appender is a plain push onto the corresponding list; the semantics of
those lists are in [`PhraseScript.cpp`](PhraseScript.cpp.md). Scripts name functions by
string, never by handle, so the binding surface stays a list of names and resolution is
deferred to call time — which is what lets a script register a precondition before the
function that implements it has been loaded.
