# src/xrGame/PhraseDialogManager.cpp

> The half of a conversation that belongs to a *participant*: the set of dialogs this character could start, the set it is currently inside, and the rule that a dialog leaves the active set the moment it stops continuing.

**Needs** — [`PhraseDialogManager.h`](PhraseDialogManager.h.md) · [`PhraseDialog.h`](PhraseDialog.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: list bookkeeping over shared dialog records; nothing here touches layout, a device or a clock.

## Purpose

A dialog has two ends. The dialog record itself (elsewhere) owns the phrase graph and
which speaker holds the floor; this file owns what one *participant* knows about dialogs —
which ones it can offer a given partner right now, and which ones it is currently engaged
in. Both the player's actor and every conversational creature inherit this, which is why
it is a mixin rather than part of either class: the trade screen and the talk screen drive
the same two objects through the same four operations.

The separation is load-bearing in one respect only: a dialog record is *shared* between
the two participants. Initiating a dialog inserts the same record into both sides' active
lists, and each side's phrase choices mutate the one record.

## State

```text
RECORD PhraseDialogManager
  checked_dialogs   : list<text>          # dialog ids already considered this availability pass
  active_dialogs    : list<Dialog>        # dialogs this participant is currently inside
  available_dialogs : list<Dialog>        # dialogs this participant could start with the current partner
```

**Invariants** — `available_dialogs` is kept sorted by descending dialog priority, and the
UI relies on that order being the presentation order, so sorting is not a convenience.
`checked_dialogs` exists only to make the availability pass idempotent within itself: the
same dialog id offered twice in one pass is silently dropped, because dialog ids reach the
pass from several independent sources (the partner's character profile, the partner's
faction, scripts) and duplicates are normal authored data, not an error. Nothing clears
it here — the subclass that runs the pass owns its lifetime.

A dialog in `active_dialogs` appears at most once; inserting a duplicate is a hard failure
rather than a merge, because two entries would double-advance the shared record.

## `GetDialogByID` / `HaveAvailableDialog`

**Contract** — linear lookup of an available dialog by its authored identifier.
`HaveAvailableDialog` answers whether one exists; `GetDialogByID` requires that it does and
returns it. The lists are short — a handful of dialogs per character — so a linear scan is
the right shape and an index would be dead weight.

**Notes** — the lookup's failure path returns the first element rather than nothing; that
is unreachable given the precondition and is an artifact of a language where returning a
reference leaves no way to say "none". A rebuild returns an optional and deletes the
precondition assertion.

## `InitDialog`

**Contract** — begins a shared dialog between this participant and a partner. The dialog
record is told who speaks and who listens, then registered as active on *both* sides.
Registering only one side would leave the partner unable to answer.

```text
FUNCTION init_dialog(partner, dialog)
  dialog.init(speaker = self, listener = partner)
  self.add_dialog(dialog)
  partner.add_dialog(dialog)
```

## `AddDialog`

**Contract** — appends a dialog to the active set, failing if it is already there.

## `ReceivePhrase`

**Contract** — the notification hook fired on a participant when the other side has said
something. The base does nothing; it exists so the actor can raise the talk screen and a
creature can choose an answer. A rebuild makes this an event the participant subscribes to.

## `SayPhrase`

**Contract** — this participant utters one phrase of an active dialog. Fails if the dialog
is not active or if it is not this participant's turn. Delegates the phrase's own effects
(script actions, information transfer, advancing the graph) to the dialog record, which
answers whether the conversation continues.

```text
FUNCTION say_phrase(dialog, phrase_id)
  REQUIRE dialog IN active_dialogs
  REQUIRE dialog says it is our turn
  continues = dialog.say_phrase(phrase_id)      # runs the phrase's actions, advances the graph
  IF NOT continues
      remove dialog from active_dialogs
```

**Invariants** — a dialog leaves the active set exactly when the shared record reports it
finished, and only on the side that spoke the terminal phrase. The partner drops it on its
own next turn. This asymmetry is why a finished dialog can briefly be active on one side
only, and no code may assume the two lists agree.

## `UpdateAvailableDialogs`

**Contract** — the hook a subclass overrides to rebuild `available_dialogs` against a
specific partner. The base implementation only sorts what is already there by descending
priority. Priority is authored per dialog and decides what the player sees first.

## `AddAvailableDialog`

**Contract** — considers one authored dialog id for a partner. Loads the dialog from
configuration, evaluates its precondition against both speakers, and keeps it only if the
precondition passes. Returns whether it was kept. Skips ids already considered in this pass.

```text
FUNCTION add_available_dialog(dialog_id, partner) -> bool
  IF dialog_id IN checked_dialogs THEN RETURN false
  append dialog_id TO checked_dialogs
  dialog = load_dialog(dialog_id)               # from the authored dialog data
  ok = dialog.precondition(self AS game object, partner AS game object)
  IF ok THEN append dialog TO available_dialogs
  RETURN ok
```

**Notes** — the dialog is loaded *before* the precondition runs, so a dialog whose
precondition always fails still costs a load on every availability pass. That is a real
cost in the shipped data (hundreds of dialogs per faction) and a rebuild is free to cache
loaded dialogs by id; nothing here depends on a fresh copy.

Both participants are required to be game objects, because the precondition is a script
call and scripts see only the game-object facade. That requirement is what ties this
otherwise-abstract mixin to the entity layer.
