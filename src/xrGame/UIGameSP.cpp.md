# src/xrGame/UIGameSP.cpp

> The single-player game's heads-up layer: which full-screen dialog opens for which key or gameplay event, and the level-change confirmation that freezes the world while the player decides.

**Needs** — [`UIGameSP.h`](UIGameSP.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`UITimeDilator.h`](UITimeDilator.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`GameTask.h`](GameTask.h.md) · [`ui/UIActorMenu.h`](ui/UIActorMenu.h.md) · [`ui/UIPdaWnd.h`](ui/UIPdaWnd.h.md) · [`ui/UITalkWnd.h`](ui/UITalkWnd.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — reached through its declarations in [`UIGameSP.h`](UIGameSP.h.md); callers name that, not this file.
**Tier floor** — T3: dialog routing and a message box; the only hard constraint is that the level-change packet be sent exactly once.

## Purpose

There is one game-UI object per game mode, and this is the single-player one. Its job is
routing: an input action or a gameplay call arrives, and it decides which of the
full-screen dialogs (inventory/trade/upgrade/corpse-search, PDA, talk, level change) is
put on top of the input stack, with what contents and in what mode. It also owns the
mechanism by which the world slows down while a menu is open, and the confirmation
prompt for crossing a level boundary.

The dialogs themselves are elsewhere; this file is the *policy* layer — who may open
what, when, and what the world does while it is open.

## State

```text
RECORD SinglePlayerGameUI EXTENDS GameUI
  game            : ClientGame         # the single-player game mode instance
  talk_window     : Dialog             # conversation screen
  change_level_window : ChangeLevelPrompt
  objective_banner : optional<Static>  # non-none only while the objective key is held
```

**Invariant** — `objective_banner` is non-none exactly while the *scores* action's key is
physically down. It is created on key press and torn down by the per-frame check; polling
the key rather than waiting for a release event is deliberate, because the release can be
swallowed by a dialog opening over the top.

```text
RECORD ChangeLevelPrompt EXTENDS Dialog
  target_graph_vertex : int            # where the actor lands, on the game graph
  target_level_vertex : int            # and on the level graph
  target_position, target_angles   : vector3
  cancel_position, cancel_angles   : vector3
  use_cancel_position : bool           # if set, refusing teleports the actor back out
  change_allowed  : bool               # false => the prompt only explains why not
  message         : text
```

## `CUIGameSP` — dialog routing

**Contract** — constructed with the talk window and the level-change prompt already
built, because both must survive across every dialog open and close; a UI reset (a
resolution or language change) destroys and rebuilds both, since their layout came from
configuration that may have changed.

Key actions map as follows, and every one of them is refused outright if the actor is
dead, is not an inventory owner, is not the view entity, or has inventory disabled by
script — this is the single choke point for "the game has taken control away from the
player":

| Action | Effect |
|---|---|
| active jobs | open the PDA |
| map | open the PDA with the map page selected |
| contacts | open the PDA with the contacts page selected |
| inventory | open the actor menu in inventory mode |
| scores | show the active task titles as a banner while held |

The scores banner shows both the storyline and the additional task when both exist; with
only one, the second line becomes that task's *description* instead of a second title;
with none, a localized "no active task" line.

**Notes** — the handler returns "not handled" even for actions it acted on. That is a
real decision, not a bug to preserve blindly: these actions are also meaningful to
layers below, and swallowing them would break them. A rebuild that wants strict
consumption must re-check every consumer.

## `StartTrade` · `StartUpgrade` · `StartCarBody`

**Contract** — each puts the actor menu on screen with a partner and a mode: trade with a
living trader, upgrade with a mechanic, corpse-search against either a dead inventory
owner or a standalone container. The corpse-search variants refuse if any dialog is
already on top; trade and upgrade do not, because they are invoked from a dialogue
script that has already established the context.

## `StartTalk`

**Contract** — clears the objective banner and shows the talk window. A flag says whether
the player may break off the conversation; a scripted conversation that must run to its
end sets it.

## `ChangeLevel`

**Contract** — arms the level-change prompt with a destination (game-graph vertex, level
vertex, position and orientation), a fallback position for refusal, whether the change is
permitted at all, and the message to show; then shows it. Re-arming while the prompt is
already on top is ignored, so a player standing in the boundary volume does not get the
prompt rebuilt every frame.

## `ChangeLevelPrompt`

**Contract** — a message box that **pauses the simulation while it is visible** and
unpauses on hide, with the on-screen "paused" caption suppressed so the player sees only
the question. A global flag marks that the pause came from here, so that the normal pause
key cannot lift it.

```text
FUNCTION on_show(prompt)
  layout FROM ("change level" OR "change level disabled" message box template)
  size prompt TO the message box's own size      # the box decides, not the parent
  set text TO prompt.message
  pause simulation, suppress pause caption
  mark pause as prompt-owned

FUNCTION on_confirm(prompt)
  hide                                            # hide BEFORE sending: the send tears
                                                  # down the level this window lives in
  send CHANGE_LEVEL to the server carrying
      prompt.target_graph_vertex, prompt.target_level_vertex,
      prompt.target_position, prompt.target_angles

FUNCTION on_refuse(prompt)
  hide
  IF prompt.use_cancel_position THEN
    move the actor TO prompt.cancel_position, prompt.cancel_angles
    # pushes the player back out of the boundary volume, so the prompt does not
    # immediately re-arm
```

The escape/quit key is bound to refusal, and no other key reaches the prompt.

**Notes** — the show path exists twice (once as the window-visibility hook, once as the
dialog-show entry) with identical bodies, because the two entry points were introduced at
different times; a rebuild should have one.

## `StartDialog` · `StopDialog`

**Contract** — wraps the base dialog stack push/pop and, on the way through, tells the
time dilator which mode is active: inventory-mode actor menu and PDA each slow the world
by their own configured factor, anything else restores normal rate. Trade, upgrade and
corpse-search deliberately do **not** dilate: they are already conversational.

## `HideShownDialogs` · `ReinitDialogs` · `OnUIReset`

**Contract** — `HideShownDialogs` closes the actor menu, the PDA, and the talk window if
it is the top receiver, and is what a cutscene or a forced game state calls to clear the
screen. `ReinitDialogs` destroys and rebuilds the talk and level-change windows from
configuration. A UI reset does the base reset and then reinitializes, because both
windows cache layout read at construction.
