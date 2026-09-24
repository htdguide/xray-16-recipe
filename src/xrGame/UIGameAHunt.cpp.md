# src/xrGame/UIGameAHunt.cpp

> The artefact-hunt interface: the team-deathmatch screen plus a reinforcement timer and the offer to pay for an early respawn.

**Needs** — [`UIGameAHunt.h`](UIGameAHunt.h.md) · [`UIGameTDM.h`](UIGameTDM.h.md) · [`UITeamPanels.h`](UITeamPanels.h.md) · [`Level.h`](Level.h.md) · [`game_cl_artefacthunt.h`](game_cl_artefacthunt.h.md) · [`team_base_zone.h`](team_base_zone.h.md) · [`ui/UIMessageBoxEx.h`](ui/UIMessageBoxEx.h.md) · [`ui/UIMoneyIndicator.h`](ui/UIMoneyIndicator.h.md) · [`ui/UIRankIndicator.h`](ui/UIRankIndicator.h.md) · [`ui/UIHelper.h`](ui/UIHelper.h.md) · [`ui/UIXmlInit.h`](ui/UIXmlInit.h.md) · [`xrUICore/ProgressBar/UIProgressShape.h`](../xrUICore/ProgressBar/UIProgressShape.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — reached through its declarations in [`UIGameAHunt.h`](UIGameAHunt.h.md); callers name that, not this file.
**Tier floor** — T3: layout re-initialization and two forwards

## Purpose

The thinnest mode interface in the family. Everything the player sees is the team
deathmatch's, re-read from this mode's own layout files, plus two additions:

- a **reinforcement timer** — in this mode the dead do not respawn individually, they
  respawn in waves, so the player needs to see how long until the next one;
- a **prompt to buy a respawn** — the player may pay to skip that wait, and must confirm
  the price.

## State

```text
RECORD ArtefactHuntUI
  reinforcement_indicator : widget       # a progress shape, or a text widget as fallback
  buy_spawn_prompt        : message box  # rebuilt on every session change
  buy_caption             : text widget
```

## `Init` — re-initialization from two layout files

**Contract** — build on the team deathmatch's widgets by pointing them at this mode's layout,
with a documented fallback.

```text
FUNCTION init(stage)
  IF stage is 0 THEN base's stage 0; create the buy caption from the shared message layout
  IF stage is 1 THEN
    initialize the scoreboard from this mode's scoreboard layout
    read THIS mode's layout, and also the deathmatch's layout as a fallback
    reinforcement indicator = a progress shape if the layout declares one,
                              otherwise a plain text widget
    frag limit, money and rank: try this mode's layout, and on failure the deathmatch's
    team icons and scores: this mode's layout only
  IF stage is 2 THEN base's stage 2; attach the reinforcement indicator
```

**Invariants** — **the fallback to the deathmatch layout is a data-compatibility mechanism,
not a defensive habit.** Two of the three shipped games place these panels differently and
one of them omits several of them from this mode's file entirely. Every fallback in this
function is marked in the source with the name of the game whose data needs it. A rebuild
must keep the "try mine, then the deathmatch's" order for exactly these four widgets, or one
game's artefact-hunt screen loses its money, rank and frag-limit readouts.

The reinforcement indicator is likewise a **shape if the data offers one and text
otherwise**, for the same reason: one game's layout declares a circular countdown and
another's declares a number.

## `SetReinforcementTimes`

**Contract** — show the time until the next respawn wave, as a fraction if the indicator is a
shape and as a bare number if it is text.

```text
FUNCTION set_reinforcement(current, maximum)
  IF the indicator is a progress shape THEN set its position from (current, maximum)
  ELSE set its text to current
```

**Notes** — the text branch drops the maximum entirely. A player looking at a number has no
idea what it is counting down from; a player looking at the shape does. That asymmetry is
inherited from the data, not chosen here.

## `SetClGame`

**Contract** — bind to a new session and rebuild the buy-respawn prompt, wiring its
confirmation directly to the game mode's purchase handler.

**Invariants** — the prompt is destroyed and recreated on every session change, because its
callback captures the session's game mode. Reusing it across sessions leaves the callback
pointing at a dead mode.

**Notes** — the callback is attached as a direct function reference rather than through the
window's named-callback table; the table-based form is present and commented out just above
it. Either works; the direct form is one fewer indirection and one fewer name to keep in
step.

## `SetBuyMsgCaption`, `UnLoad`

**Contract** — write a string-table identifier into the buy caption; forward the teardown.

**Notes** — this layer's teardown adds nothing, because the two widgets it created are
marked for automatic deletion with their parent. The prompt is destroyed by the object's own
teardown rather than here, which means it outlives an unload and is correct only because a
new session rebuilds it.
