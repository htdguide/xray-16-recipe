# src/xrGame/AI_PhraseDialogManager.cpp

> The non-player half of a conversation: given a phrase graph the player has advanced, pick the reply that matches how much this character likes the player, and say it.

**Needs** — [`AI_PhraseDialogManager.h`](AI_PhraseDialogManager.h.md) · [`PhraseDialog.h`](PhraseDialog.h.md) · [`PhraseDialogManager.h`](PhraseDialogManager.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`GameObject.h`](GameObject.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`ui/UITalkWnd.h`](ui/UITalkWnd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a selection over a list plus one registry lookup

## Purpose

A conversation is a graph of phrases; the player picks one, and the character must pick
back. This file is that reply rule, mixed into every character who can talk. It exists
apart from the shared dialogue machinery in
[`PhraseDialogManager.cpp`](PhraseDialogManager.cpp.md) because the shared machinery is
side-neutral — it works the same for the player — and only the non-player side has to
*choose*.

The rule is that the reply a character gives is a function of its **attitude** toward the
speaker. Every authored phrase carries a goodwill threshold; the character says the
friendliest phrase whose threshold it can meet. This is why the same conversation reads
politely to a trusted player and curtly to a despised one without any authored branching
beyond the thresholds.

## State

```text
RECORD AiDialogManagerState
  start_dialog          : optional<text>   # dialogue this character opens with on meeting the player
  default_start_dialog  : optional<text>   # what the above is restored to
  pending_dialogs       : list<dialog>     # declared, never filled — see Notes
```

**Invariant** — a character who implements this must also be an inventory owner and must
be reachable as a game object. Both are asserted at every entry point rather than
checked, because a character that is neither cannot have a name, a reputation or a
partner, and the conversation has nowhere to go.

## `ReceivePhrase`

**Contract** — called when the partner has said something. Answers first, then runs the
shared receive path (which advances the graph and decides whether the conversation has
ended). The order matters: the answer must be chosen against the graph state the player's
phrase produced, and the shared path may finish the dialogue.

## `AnswerPhrase`

**Contract** — chooses and speaks one reply. No effect when the dialogue is already
finished, which is how a terminal phrase ends a conversation without a farewell. Pushes
the reply's display text and the speaker's name into the talk screen, then routes the
phrase through the shared *say* path so that the graph advances and any script callbacks
on the phrase fire.

**Invariants** — exactly one phrase is spoken per call; the phrase spoken is always drawn
from the candidate list the graph currently offers, never invented.

```text
FUNCTION answer_phrase(dialog)
  IF dialog.is_finished THEN RETURN

  me      = this as inventory owner
  partner = dialog.partner_of(me) as inventory owner
  attitude = relation_registry.attitude(from: partner, to: me)

  # The candidate list is authored in priority order, friendliest first.
  # Fall back to the LAST entry — the rudest — when nothing qualifies.
  chosen = last index of dialog.phrases
  FOR EACH i, phrase IN dialog.phrases
    IF attitude >= phrase.goodwill_threshold THEN
      chosen = i
      BREAK

  # Several phrases may share the winning threshold; pick among them at random
  # so a character repeating a conversation does not repeat itself word for word.
  tied = [ i FOR i IN dialog.phrases WHERE phrases[i].goodwill == phrases[chosen].goodwill ]
  chosen = random element of tied

  talk_screen.add_answer(dialog.text_of(phrases[chosen].id), me.name)
  say_phrase(dialog, phrases[chosen].id)
```

**Notes**

- The fallback to the last entry rather than to silence is the authored contract with the
  dialogue data: a phrase list is written rudest-last, so a character with no civil reply
  available still has *something* to say. A rebuild that sorts the list breaks every
  shipped conversation.
- The goodwill used for the tie-break is read from whichever phrase the first loop last
  examined, not from the winning phrase — in the fall-through case those differ, and the
  tie set is then drawn against the last phrase's threshold. This reads like an oversight
  but is observable in shipped dialogue, so reproduce it rather than fixing it.
- The talk screen is reached by casting the current game interface to its single-player
  form. Conversation is a single-player feature; in a multiplayer session this path is
  never entered.

## `SetStartDialog` · `SetDefaultStartDialog` · `GetStartDialog` · `RestoreDefaultStartDialog`

**Contract** — the character's opening dialogue is two values: the one currently in force
and the one authored. Scripts override the first to stage a quest conversation and then
restore it. Setting the default does not change what is in force.

## `UpdateAvailableDialogs`

**Contract** — rebuilds the set of conversations this character can offer a given
partner. Clears both the available and the already-checked sets, unconditionally offers
the character's current start dialogue (when it has one) and the universal greeting
dialogue, then lets the shared path add every dialogue whose preconditions pass against
this partner.

**Notes** — the greeting dialogue is named by a fixed identifier that must exist in the
game data. Its presence is what guarantees every character is talkable at all, even one
with no authored conversations.
