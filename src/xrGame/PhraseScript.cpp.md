# src/xrGame/PhraseScript.cpp

> The script and information-portion gate attached to every phrase and every dialog: the predicates that decide whether a line may be said, and the effects that fire when it is.

**Needs** — [`PhraseScript.h`](PhraseScript.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`InfoPortion.h`](InfoPortion.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`Actor.h`](Actor.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — reached through its declarations in [`PhraseScript.h`](PhraseScript.h.md); callers name that, not this file.
**Tier floor** — T3: string lists, an XML read and script calls

## Purpose

Every node of the conversation system — a whole dialog, and each phrase within it — carries
the same small bundle: some predicates that must hold before it is offered, and some
effects that run once it is chosen. This file is that bundle, written once and embedded in
both, which is why it is a separate class rather than fields on either.

Two kinds of condition live side by side and they are not interchangeable. *Information
portions* are the game's own persistent flags — the quest state — and are checked and set
directly. *Script functions* are named by string and resolved at call time. Authors reach
for the first for anything the save file must remember and the second for anything
computed.

## State

```text
RECORD PhraseScript
  script_text_func : text          # optional: a script function that supplies display text
  actions          : list<text>    # script function names, fired in order when chosen
  preconditions    : list<text>    # script predicate names, all must pass
  has_info         : list<text>    # information portions the actor must hold
  dont_has_info    : list<text>    # information portions the actor must not hold
  give_info        : list<text>    # portions granted when chosen
  disable_info     : list<text>    # portions revoked when chosen
```

**Invariants** — all seven lists are ordered and the order of `actions` is observable,
since two actions may touch the same world state. The information checks are always
evaluated against **the actor**, never against the speaker whose bundle this is — see the
note under `CheckInfo`, because this is the single most surprising fact in the dialog
system.

## `Load`

**Contract** — reads the bundle out of one XML node of a dialog file. Each list is the
text content of every child element with the matching tag name, in document order:
`precondition`, `action`, `has_info`, `dont_has_info`, `give_info`, `disable_info`. Absent
tags give empty lists. Replaces whatever was there; loading twice does not accumulate.

**Notes** — the per-tag read is one shared step rather than six copies because the only
difference between the six is the tag name and the destination.

## `CheckInfo`

**Contract** — passes when the actor holds every portion in `has_info` and none in
`dont_has_info`. Short-circuits on the first failure. No side effects.

**Notes** — the parameter is the *owner whose bundle this is*, but the lookups go to the
player's actor regardless. This is deliberate and it is what the shipped dialog data
assumes: the quest flags a conversation branches on are the player's, and a creature's own
knowledge is not modelled as information portions at all. The parameter survives only to
name the speaker in diagnostics. A rebuild should make the actor an explicit argument
rather than a global reach, but must not "fix" the semantics to check the speaker — every
shipped dialog would change.

## `TransferInfo`

**Contract** — grants every portion in `give_info` and revokes every one in `disable_info`,
in that order, on the actor. Granting is what fires the information-portion callbacks that
advance quests, so this is the point where saying a line changes the world.

## `GetScriptText`

**Contract** — given the phrase's authored text, returns either that text unchanged, or —
when a text function is configured — whatever that script function returns for this pair
of speakers, this dialog and this phrase. Resolving a configured-but-missing function is a
hard failure, not a silent fallback, because a dialog rendering the raw identifier is worse
than a crash at load.

```text
FUNCTION get_script_text(authored_text, speaker, partner, dialog_id, phrase_id) -> text
  IF script_text_func is empty THEN RETURN authored_text
  fn = resolve script function script_text_func      # FAIL WITH "cannot find phrase script text"
  RETURN fn(speaker AS game object, partner AS game object, dialog_id, phrase_id)
```

**Notes** — the returned text is a borrowed string owned by the script runtime. It must be
consumed before the next script call that could collect it; the original relies on the
caller copying it into the UI immediately. A rebuild that returns an owned string deletes
the hazard.

## `Precondition`

**Contract** — two forms, differing only in how many speakers the predicate is handed.

The **one-speaker form** is the dialog-level gate: is this whole conversation offerable?
The **two-speaker form** is the phrase-level gate, and additionally names the phrase being
answered and the candidate next phrase, so a predicate can reason about the branch it is
about to enable.

Both first run the information check and reject immediately if it fails; then evaluate
each script predicate in order, stopping at the first that returns false. An unresolvable
predicate name is a hard failure. No side effects in either form — a precondition that
mutates the world is an authoring bug the engine does not defend against.

```text
FUNCTION precondition(speaker, partner?, dialog_id, phrase_id, next_phrase_id?) -> bool
  IF NOT check_info(speaker) THEN RETURN false
  FOR EACH name IN preconditions
      fn = resolve script predicate name           # FAIL WITH "cannot find precondition"
      IF NOT fn(<the speakers and ids for this form>) THEN RETURN false
  RETURN true
```

**Notes** — the two argument shapes are not a convenience: the script functions authored
for dialogs and those authored for phrases have different signatures in the shipped data,
and picking the wrong one silently passes the wrong values. A rebuild keeps them as two
distinct operations.

The rejection reasons are logged only under a dialog debug flag. Dialog authoring is
otherwise unfalsifiable — a phrase that never appears gives no clue why — so this logging
is the tool that makes the system workable and should survive.

## `Action`

**Contract** — runs the bundle's effects after a line is chosen. Two forms again, matching
the two precondition forms, and they differ in more than arity:

- the **one-speaker form** calls each script action, then transfers information;
- the **two-speaker form** transfers information *first*, then calls each script action,
  and swallows any error a script action raises.

```text
FUNCTION action(speaker, dialog_id, phrase_id)            # dialog level
  FOR EACH name IN actions: resolve(name)(speaker AS game object, dialog_id)
  transfer_info(speaker)

FUNCTION action(speaker, partner, dialog_id, phrase_id)   # phrase level
  transfer_info(speaker)
  FOR EACH name IN actions
      fn = resolve(name)
      TRY fn(speaker AS game object, partner AS game object, dialog_id, phrase_id)
      ON ERROR: continue                                  # see note
```

**Invariants** — in the phrase form the information transfer happens *before* the script
actions, so a script action can observe the portions its own phrase just granted. That
ordering is relied on by shipped quest scripts and is not interchangeable with the dialog
form's ordering.

**Notes** — the phrase form absorbs script errors and continues to the next action. This
is a deliberate asymmetry: a phrase action runs mid-conversation with a UI screen open and
two entities mid-exchange, and letting a modder's faulty script tear that down strands the
player in a half-closed dialog. The dialog form runs at conversation start, where failing
loudly is recoverable. The cost is that a broken action is invisible; a rebuild should
still contain the failure, but log it.
