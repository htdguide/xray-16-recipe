# src/xrGame/game_cl_capture_the_artefact_captions_manager.cpp

> Every on-screen prompt in capture the artefact, decided in one place each update: clear everything, then re-assert whatever the current phase and player state call for.

**Needs** — [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md) · [`game_cl_capture_the_artefact.h`](game_cl_capture_the_artefact.h.md) · [`UIGameCTA.h`](UIGameCTA.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`ui/TeamInfo.h`](ui/TeamInfo.h.md)
**Used by** — reached through its declarations in [`game_cl_capture_the_artefact_captions_manager.h`](game_cl_capture_the_artefact_captions_manager.h.md); callers name that, not this file.
**Tier floor** — T3: string assembly and a countdown against the server clock

## Purpose

Prompts like "press B to buy" and "press jump to pay for a spawn" have many independent
causes and one screen. Written directly from each cause, they stick: a prompt set by a
condition that later stops holding is never unset, because nothing remembers who set it.

This object solves that by inverting the flow. Its owner *tells it facts* — buying is
possible now, the paid spawn is affordable now, this team won — and once per update it
**clears every caption and re-derives all of them** from the facts plus the current phase.
Nothing has to be unset, because everything is unset every frame.

It is a separate object from the mode because the derivation is a decision tree of its own,
and separating it keeps the mode's update loop about rules rather than about text.

## State

```text
RECORD CaptionsManager
  game        : CaptureTheArtefactClient     # the mode; source of phase and player state
  game_ui     : CaptureTheArtefactUI         # the screen the captions land on
  can_show_buy      : bool                   # pushed in: the player may buy right now
  can_show_payspawn : bool                   # pushed in: the paid spawn is affordable
  can_spawn         : bool                   # pushed in; nothing reads it
  winner_team       : Team                   # spectator value = "no winner yet"
  warmup_message    : text                   # rebuilt by the warm-up countdown
  timelimit_message : text                   # rebuilt by the time-limit countdown
  last_seconds_left : int                    # so the countdown announcement fires once per second
```

Invariants:

- **The three pushed flags are inputs, not conclusions.** They describe the world; what
  appears on screen is derived from them together with the phase and the player's state.
- **`winner_team` holding the spectator value means "no winner"**, and the score phase
  asserts it has been set. A rebuild should use an explicit optional.
- **`last_seconds_left` starts at ten**, above the five-second announcement window, so no
  announcement fires on the first update after initialisation.

## `ShowCaptions`

**Contract** — the once-per-update pass. Clears every caption, then dispatches on the mode's
phase. Does nothing before the owner has bound it.

```text
FUNCTION show_captions()
  IF not bound: RETURN
  clear every caption
  SWITCH game.phase
    PENDING:       show_pending_captions()      # nothing
    IN_PROGRESS:   show_in_progress_captions()
    PLAYER_SCORES: show_score_captions()
```

**Notes** — the clear-then-reassert shape is the whole design and it is worth copying. Its
cost is that every caption is written every update even when unchanged, which the screen
absorbs; its benefit is that no caption can outlive its reason.

The pending phase deliberately shows nothing. That is a decision, not a gap: before the round
starts the screen belongs to the team panels and the server information.

## `ShowInProgressCaptions`

**Contract** — the live-round decision tree. Assembles the warm-up and time-limit captions if
those are running, then branches on what the player *is*: a spectator, a demo viewer, a live
player, or a dead one.

```text
FUNCTION show_in_progress_captions()
  ps = my player state ; IF none or skipped: RETURN
  IF warming up:       caption(warm-up) = warmup_message
  IF a time limit:     caption(time)    = timelimit_message

  entity = what I am controlling ; IF none: RETURN

  IF I am on the spectator team
    prompt "press jump to select a team"
    caption(spectator) = the spectator's own description of what it is watching
    RETURN
  IF a demo is playing
    caption(spectator) = the same description
    RETURN

  IF can_show_buy: prompt(buy) = "press B to buy"

  IF I am permanently dead
    IF I am still controlling an ACTOR:   prompt "press fire for spectator"
    ELSE IF can_show_payspawn:            prompt "press jump to pay for a spawn"
```

**Notes** — the dead branch splits on *what the player is controlling*, not on how long he
has been dead, and that is the load-bearing distinction. A freshly killed player is still
attached to his corpse and is told how to detach; once detached into the spectator camera he
is told how to get back in. Two prompts, one state, distinguished by the camera.

The spectator's own text (who it is following, in what mode) is asked of the spectator rather
than assembled here, because only the spectator knows its own cycle position.

Note that the "press jump to select a team" prompt is given to anyone on the spectator team,
including someone who chose to spectate deliberately. There is no way to tell the two apart
at this level.

## `ShowScoreCaptions`

**Contract** — the end-of-round caption: "<team> wins", with the team name localised and the
sentence formatted from the string table. Asserts a winner has been set.

**Notes** — the team name lookup takes the winning team **plus one**, because the team-name
table is one-based while this mode's team values are not. That offset is the numbering hazard
the mode itself warns about, surfacing here as a magic `+ 1`.

## `SetWarmupTime`

**Contract** — rebuilds the warm-up caption from the deadline and the current server time,
and **returns a number of seconds** when a countdown announcement should be spoken this
update — one to five — or zero. Clamps a passed deadline to now, so the remaining time never
underflows.

```text
FUNCTION set_warmup_time(deadline, now) -> seconds_to_announce
  IF now >= deadline: deadline = now              # never negative
  remaining = deadline - now
  announce = 0

  IF remaining > 10 seconds
    caption = "time to start" + " " + hh:mm:ss
  ELSE IF remaining < 1 second
    caption = "go"
  ELSE
    seconds = remaining / 1000
    IF seconds != last_seconds AND 0 < seconds <= 5
      announce = seconds                          # fire the spoken countdown once
    last_seconds = seconds
    caption = "ready" + "..." + seconds

  RETURN announce
```

**Invariants** — the announcement fires **once per whole second**, guarded by comparing
against the previous update's second. Without that guard the countdown would be spoken every
frame.

**Notes** — three display regimes, at ten seconds and one second. Above ten seconds it is a
clock; between ten and one it is a bare number with "ready"; below one it is "go". The
spoken countdown covers only the last five, so seconds ten to six are shown and not spoken.

Returning the number to announce rather than announcing it is the right split: this object
owns text, and the mode owns sound.

The caption buffer is explicitly emptied before assembly, which the source itself annotates
as bad style. A rebuild building a string rather than concatenating into a fixed buffer
needs none of it.

## `SetTimeLimit`

**Contract** — rebuilds the time-limit caption as a remaining clock, or writes a zero clock
straight to the screen once the limit has passed.

**Notes** — asymmetric: the live case writes the member that the in-progress pass will pick
up, and the expired case writes the screen **directly**, bypassing the clear-and-reassert
cycle. The result is the same on screen but the two paths disagree about who owns the
caption; a rebuild should write the member in both cases.

## `ConvertTime2String`

**Contract** — milliseconds to a zero-padded hours:minutes:seconds string. Truncates rather
than rounds.

## `Init` / `ResetCaptions` / `CanCallBuySpawn` / `CanCallBuy` / `CanSpawn` / `SetWinnerTeam`

**Contract** — bind to the mode and its screen; clear every caption; and the four setters
through which the mode pushes facts in.

**Notes** — the spawn-possible setter is stored and never read. It is a fact nothing derives
a caption from any more; the prompt it once fed is commented out at its call site. A rebuild
should drop it.

The buy setter tolerates not being bound yet and forces its flag false; the other two assert
instead. That inconsistency has no reason beyond the order in which they were written.
