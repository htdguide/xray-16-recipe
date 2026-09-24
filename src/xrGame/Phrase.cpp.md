# src/xrGame/Phrase.cpp

> One line of a conversation: its text, the goodwill it demands, and whether it is a dead end.

**Needs** — [`Phrase.h`](Phrase.h.md) · [`PhraseScript.h`](PhraseScript.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a value type with three predicates

## Purpose

A phrase is a vertex of a dialog graph. It carries what to display, what the speaker must be
liked well enough to say it, and the script hooks that decide whether it is offered at all
and what happens when it is said. Almost none of that is code — the file exists because the
*text* of a phrase has three possible sources and something has to resolve between them.

## State

```text
RECORD Phrase
  id             : text     # unique within one dialog; "0" is the dialog's entry phrase
  text           : text     # a string-table key, or empty
  script_text_id : text     # name of a script function returning the text, or empty
  script_text_val: text     # cache of that function's last result
  goodwill_level : int      # minimum standing the listener must hold toward the speaker
  is_finalizer   : bool     # saying this ends the conversation regardless of outgoing edges
  script_helper  : DialogScriptHelper   # preconditions, effects, and the text transform
```

**Invariant** — `script_text_val` is meaningful only immediately after the dialog has asked
for this phrase's text; it is a scratch slot on a shared record, not part of the phrase's
identity. That is the one sharp edge here: phrase records are shared across every live
conversation using the same dialog, so a rebuild that evaluates two conversations of the
same dialog concurrently must not keep the result on the phrase.

## `GetText` / `GetScriptText`

**Contract** — return the authored text and the last script-produced text respectively. Both
are plain reads; the choice between them belongs to the dialog, not the phrase.

## `IsDummy`

**Contract** — true when the phrase has no text from any of its three sources: no authored
text, no script text identifier, and no cached script result. A dummy phrase is a graph node
that exists only to carry edges and script effects, and the conversation UI must not offer
it as something to click. The dialog uses this to decide whether an entire set of available
replies is invisible, which is the difference between "the conversation continues silently"
and "the player is shown an empty list".

**Notes** — all three sources are tested, not just the authored one, because a phrase whose
text comes from a script has empty authored text and would otherwise read as a dummy before
its script has ever run.
