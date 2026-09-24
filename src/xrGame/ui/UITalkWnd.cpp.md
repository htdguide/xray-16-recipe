# src/xrGame/ui/UITalkWnd.cpp

> The conversation: it holds the two speakers and the active phrase graph, turns a clicked question
> into a spoken phrase, and hands off to trade or upgrade.

**Needs** — [`UITalkWnd.h`](UITalkWnd.h.md) · [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`../UIGameSP.h`](../UIGameSP.h.md) · [`../Actor.h`](../Actor.h.md) · [`../PDA.h`](../PDA.h.md) · [`../trade.h`](../trade.h.md) · [`../Level.h`](../Level.h.md) · [`../PhraseDialog.h`](../PhraseDialog.h.md) · [`../PhraseDialogManager.h`](../PhraseDialogManager.h.md) · [`../game_cl_base.h`](../game_cl_base.h.md) · [`../../xrServerEntities/character_info.h`](../../xrServerEntities/character_info.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`UITalkWnd.h`](UITalkWnd.h.md)
**Tier floor** — T2: owns a sound handle whose position is updated per frame and whose release is
ordered against the screen closing

## Purpose

A conversation is a walk over an authored **phrase graph**: the other party offers dialogs, the
player picks one, and from then on each side's available phrases are whatever the graph says they
are at the current node, filtered by preconditions. This file drives that walk from the screen and
does three further things the graph does not: it plays the voice line for each phrase, it turns the
camera toward the other speaker, and it hands the pair off to the trade or upgrade screen.

It owns no widgets. Every visual act goes through [`CUITalkDialogWnd`](UITalkDialogWnd.cpp.md),
which reports back by notification.

## State

```text
RECORD Conversation extends ModalDialog
  view             : DialogueScreen        # the visual half; a child, auto-deleted
  actor            : optional<Actor>       # the player, while talking
  ours, theirs     : InventoryOwner        # the two speakers
  our_manager,
  their_manager    : DialogManager         # each side's view of the phrase graph
  current_dialog   : optional<PhraseDialog># none means TOPIC MODE: pick a subject
  questions_stale  : bool                  # recompute on the next frame
  disable_break    : bool                  # the player may not walk away
  voice            : SoundHandle           # at most one line playing
```

Invariants:

- **`current_dialog` absent is a mode, not an error.** Absent means the player is choosing a
  subject, and the question list shows the available dialogs; present means the player is inside one
  and the list shows that node's phrases. Every path that finishes a dialog must return to the
  absent state, and there is one function that does it.
- Question refresh is **deferred**: handlers set a flag, the next update recomputes. This is what
  makes it safe to refill the question list from inside a question's own click handler.
- At most one voice line plays; starting one stops the previous.
- The screen is a full-canvas window, so its coordinates are the canvas's.

## `InitTalkWnd`

**Contract** — Sizes the screen to the whole virtual canvas, builds the visual half as an
auto-deleted child, and points it back at this object so it can request a stop.

## `Show`

**Contract** — Opening initialises the conversation and announces the information portion
`ui_talk_show` to scripts. Closing stops the voice, hides the visual half, announces `ui_talk_hide`,
returns to topic mode, and ends the player character's talking state — but only if this screen
started it.

```text
FUNCTION Show(open)
  base.Show(open)
  IF open THEN
    InitTalkDialog()
    announce to scripts: "ui_talk_show"
  ELSE
    StopSnd()
    view.Hide()
    announce to scripts: "ui_talk_hide"
    IF actor EXISTS THEN
      to_topic_mode()
      IF actor is still talking THEN actor.stop_talking()
      actor <- none
```

## `InitTalkDialog`

**Contract** — Resolves both speakers from the player character's current talk partner, wires each
side's dialog manager, fills both portraits, sets both names, clears the log, plays the other
party's opening line, marks the questions stale, runs one update immediately, then configures and
shows the visual half. Returns without doing anything if the player is not actually talking.

```text
FUNCTION InitTalkDialog()
  actor <- the player character
  IF actor EXISTS AND actor is not talking THEN RETURN      # nothing to show

  ours   <- actor as an inventory owner
  theirs <- actor.talk_partner
  our_manager, their_manager <- each as a dialog manager

  view.our_portrait.load(ours.id); view.others_portrait.load(theirs.id)
  view.SetOurName(ours.name);      view.SetOthersName(theirs.name)
  view.ClearAll()

  InitOthersStartDialog()
  questions_stale <- true
  Update()

  view.mechanic_mode <- theirs is an upgrade mechanic
  view.SetOsoznanieMode(theirs requests the stripped presentation)
  view.Show()
  view.UpdateButtonsLayout(disable_break, theirs.trade_enabled)
```

**Notes** — The guard reads oddly — it returns when the player exists *and* is not talking, and
proceeds when there is no player at all. That is the shipped behaviour; a rebuild that also returns
on a missing player is strictly safer and changes nothing observable, since the next line would
otherwise fail.

## `InitOthersStartDialog`

**Contract** — Asks the other party's manager to recompute which dialogs it may offer; if any, takes
the first, opens it, speaks its phrase `"0"` — the conventional root phrase of every dialog — and,
if that phrase already finished the dialog, returns to topic mode.

**Notes** — The identifier `"0"` as a dialog's entry phrase is a convention of the shipped dialog
data, not something the graph format enforces. It appears in three places in this file and one in
the dialog manager.

## `UpdateQuestions`

**Contract** — Refills the question list for the current mode, then clears the stale flag. The
densest decision in the file.

```text
FUNCTION UpdateQuestions()
  view.ClearQuestions()

  IF current_dialog IS none THEN                     # TOPIC MODE
    our_manager.recompute_available_dialogs(their_manager)
    FOR EACH dialog, index IN our_manager.available_dialogs
      AddQuestion(dialog.caption, dialog.id, index,
                  is_finalizer = dialog.phrase("0").is_finalizer)
  ELSE IF current_dialog.we_are_speaking(our_manager) THEN
    IF current_dialog has phrases AND every one of them is a dummy THEN
      say a uniformly random one            # the player makes no choice here
    IF current_dialog EXISTS AND not every phrase is a dummy THEN
      FOR EACH phrase, index IN current_dialog.phrases
        AddQuestion(text of phrase, phrase.id, index, phrase.is_finalizer)
    ELSE
      UpdateQuestions()                     # re-enter: the dialog changed under us
  IF the input device is a controller THEN view.FocusOnFirstQuestion()
  questions_stale <- false
```

**Invariants** — A node whose phrases are all *dummies* has no player choice in it: the conversation
speaks one at random and moves on. This is how an authored graph makes a character monologue without
the player clicking through it. Saying the phrase can finish the dialog or advance it, so the
function re-enters to recompute from the new node — the one recursion here, and it terminates
because a dialog cannot stay at an all-dummy node after speaking.

When nothing is speakable at this node (the other party's turn), the question list is simply left
empty and the player waits for the graph to advance.

## `AskQuestion`

**Contract** — Runs when the visual half reports a question click. In topic mode, resolves the
clicked identifier to a dialog, opens it and speaks its root phrase; inside a dialog, speaks the
clicked phrase. Marks the questions stale rather than refreshing them, so the refresh happens after
the handler returns.

Refuses to run when the questions are already stale — which is the guard against a fast double click
firing twice against a list that has not been rebuilt yet.

```text
FUNCTION AskQuestion()
  IF questions_stale THEN RETURN          # a second click before the refresh: ignore
  IF current_dialog IS none THEN
    VERIFY the clicked id names an available dialog
    current_dialog <- our_manager.dialog_by_id(clicked_id)
    our_manager.open(their_manager, current_dialog)
    phrase <- "0"
  ELSE
    phrase <- clicked_id
  SayPhrase(phrase)
  questions_stale <- true
```

## `SayPhrase`

**Contract** — Logs the phrase as an answer from the player, tells the manager the phrase was said —
which is what actually advances the graph and fires any script effects — and returns to topic mode
if the dialog finished.

## `AddAnswer`

**Contract** — Drops an empty phrase entirely, starts the voice line, translates the text, and
appends it to the log attributed to the given speaker.

**Speaker identity is decided by comparing names**, which the source itself flags as unreliable when
both parties happen to share a name. The comparison only chooses which of two log templates and
which portrait the news entry uses, so a collision is cosmetic.

## `AddQuestion`

**Contract** — Drops an empty phrase, translates the text, and forwards to the visual half with the
phrase identifier and the finalizer flag.

## `Update`

**Contract** — Per frame: end the conversation if the player stopped talking; close the screen
outright if either speaker stopped being a live object; re-show the visual half if this screen is
the top input receiver and the visual half somehow is not shown; refresh the questions if stale;
update the camera; refresh the button layout; and keep the voice line positioned on the speaker.

```text
FUNCTION Update()
  IF the player exists AND is no longer talking THEN StopTalk()
  ELSE IF either speaker is no longer a live object THEN hide the screen

  IF this screen is the top input receiver AND view is hidden THEN view.Show()
  IF questions_stale THEN UpdateQuestions()
  base.Update()
  point the camera at the other speaker
  view.UpdateButtonsLayout(disable_break, theirs.trade_enabled)
  IF a voice line is playing THEN
    move the sound to the other speaker's position, raised 1.8 units
```

**Notes** — The 1.8-unit raise is a stand-in for the speaker's head. The source marks it twice as
something that should track the actual head bone; it is a constant because the sound is positioned
from the object's origin, which is at the feet. A rebuild with access to the skeleton should use the
head bone and will sound slightly different — better — than the original.

## The camera turn

**Contract** — Each frame, if the camera is more than about 0.2 radians off the direction to the
other speaker's centre — raised by half their radius, so it looks at the chest rather than the feet
— the camera's yaw and pitch each ease toward it with a bounded angular rate. Each axis is tested
and eased independently, so the camera may be turning in one and already settled in the other.

**Notes** — The easing parameters (a 0.15 factor, a 0.2 threshold, a cap of thirty degrees per step)
are tuning with no derivation in the source. They are what makes the turn feel like a head turn
rather than a snap.

## `PlaySnd` / `StopSnd`

**Contract** — A phrase's voice line is found **by the phrase text itself**: the localization
identifier of the phrase is also the file name, under a fixed directory and extension. If no such
file exists, nothing plays and the conversation continues silently.

Before playing, the player character is offered the line through a dialog-sound hook; if the
character handles it (a scripted sequence taking over the audio), this screen does not play it.
Stopping offers the same hook first.

```text
FUNCTION PlaySnd(text)
  IF text IS empty THEN RETURN
  path <- "characters_voice/dialogs/" + text truncated to fit + ".ogg"
  StopSnd()
  IF the sound file does not exist THEN RETURN
  IF the player character handles this line itself THEN RETURN
  voice <- create an effect sound at the other speaker's position, raised 1.8 units
  play it
```

**Notes** — Deriving the file name from the localization identifier is why dialog identifiers in the
shipped data look like file paths. The truncation is to the path buffer, so a very long identifier
silently addresses a different file — a hazard the shipped data avoids by convention.

## `SwitchToTrade` / `SwitchToUpgrade`

**Contract** — Both hide the visual half, stop the voice, and ask the single-player game UI to open
the corresponding screen for the two speakers. Trade additionally requires that **both** parties
have trading enabled; upgrade has no such check in the shipped code — the equivalent test is present
but commented out, so an upgrade is offered whenever the other party is a mechanic.

## `OnKeyboardAction`

**Contract** — On a press: use, quit and the UI back action all leave the conversation, unless
breaking off is disabled. The talk-to-trade action switches to trade or upgrade depending on the
mechanic flag, and is refused during the stripped presentation. On a hold: the two log-scroll
actions step the answer log.

On press **or** hold: the UI up and down actions move the question focus — and the wrap flag is
`press, not hold`, which is how holding a direction walks to the end of the list and stops while
tapping cycles.

## `OnControllerAction`

**Contract** — The controller's UI-move axis moves the question focus by the sign of its vertical
component, and the log-scroll axis scrolls the log the same way. Same wrap rule as the keyboard.

## `StopTalk` / `Stop`

**Contract** — `StopTalk` hides the screen, which runs the whole closing path. `Stop` is empty: it
is the deferred-stop hook the declaration promises and nothing uses. A rebuild may drop it; nothing
calls it.
