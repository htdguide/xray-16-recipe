# src/xrGame/autosave_manager.cpp

> Periodically saves the game by itself, but only at a moment when saving is safe and the result is worth loading.

**Needs** — [`autosave_manager.h`](autosave_manager.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`ai_space.h`](ai_space.h.md) · [`date_time.h`](date_time.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md)
**Used by** — reached through its declarations in [`autosave_manager.h`](autosave_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a timer on the scheduler, a save request and a screenshot; the file-attribute call is the only platform contact

## Purpose

An unattended save every so often. The interesting part is not the timer but the refusal
rules: a save taken while the level is still streaming in, while the actor does not exist
yet, while the player is dead, or while any subsystem has declared itself mid-transaction,
produces a save file that either fails to load or loads into a broken world. So the manager
does not save on a schedule; it *offers* to save on a schedule and defers when anything
objects.

It also has a second job that is easy to miss: the save's thumbnail. A save the player will
later pick out of a list needs a picture of where they were, and the only moment that
picture can be taken is the moment of the save.

## State

```text
RECORD CAutosaveManager
  autosave_interval       : int (milliseconds)   # from configuration, as h:m:s
  delay_autosave_interval : int (milliseconds)   # from configuration, as h:m:s
  last_autosave_time      : int (milliseconds)   # global clock
  not_ready_count         : int                  # how many subsystems currently forbid saving
```

**Invariants**

- `not_ready_count` is a counter, not a flag, because several subsystems can independently
  forbid saving at the same time and each must be able to lift its own veto without
  clearing anyone else's. Every increment must be matched by exactly one decrement;
  decrementing at zero is a bug in the caller and is asserted rather than clamped, because
  an under-count silently re-enables saving inside somebody's transaction.
- Deferral is expressed by *pushing the last-save timestamp forward*, not by holding a
  separate retry time. That is what makes an indefinitely-blocked save retry at the delay
  interval rather than every scheduler tick.
- Both intervals are authored as clock strings and converted to a duration. The manager runs
  on the real-time clock, not the game clock, so a long in-game time skip does not trigger a
  burst of saves.

## scheduling

**Contract** — the manager registers itself with the scheduler at construction with a fixed
update period of five seconds in both directions — a minimum and a maximum that are equal,
which pins it to exactly that rate regardless of load. Its load-scale factor is a half,
which is the scheduler's hint that this is cheap and may be favoured. It unregisters at
destruction. It always declares itself needing an update, so it is never skipped.

**Notes** — five seconds is the granularity of the whole feature. The configured interval is
therefore effectively rounded up to the next multiple of five seconds; nothing depends on
exactness.

## `shedule_Update`

**Contract** — the whole feature in one path. Runs every five seconds. Saves nothing unless
the alife simulation exists, the interval has elapsed, and every readiness condition holds;
otherwise defers.

```text
FUNCTION scheduled_update(dt)
  IF the alife simulation is not running THEN RETURN      # no alife, no save format
  IF last_autosave_time + autosave_interval >= now THEN RETURN

  IF level is still precaching
     OR there is no actor
     OR not_ready_count > 0
     OR the actor is dead THEN
    last_autosave_time = last_autosave_time + delay_autosave_interval   # try again later
    RETURN

  last_autosave_time = now

  name = "<user name> - autosave"
  request a save under that name, without the alife-time flag

  take a screenshot sized for a save thumbnail into $game_saves$/<name>.dds
  hide that file from the user's file browser where the platform can
  show the on-screen "autosave" notice
```

**Invariants** — the precache check is the important one. During precaching the level's
objects exist but their visuals, sounds and physics are still being brought up; a save taken
there records an inconsistent world. The dead-actor check is a different kind: saving over
the player's last autosave at the moment they die leaves them with nothing to load back to.

**Notes** — the save is issued as a *message to the authoritative side* rather than a direct
call, so that the save happens at the point in the frame where the world is quiescent,
through exactly the same path as a save the player asked for. There is only one save
implementation and the autosave does not get a private one.

**Notes** — the save name is derived from the user's profile name, so two players on one
machine do not overwrite each other's autosave, and there is exactly one autosave slot per
player — each autosave replaces the last. A rolling set of slots would be a data-loss
improvement and is not what shipped.

**Notes** — the thumbnail is hidden at the filesystem level on the platform that has such an
attribute. It is an implementation detail of the save list, not a file the player should
see beside their saves; a rebuild can equally keep thumbnails in a subdirectory or inside
the save file.

**Notes** — the on-screen notice is shown for three seconds in the two older games' data and
for an unbounded time (until something else replaces it) in the newest, because the newer
data drives its own timeout from the notice's own definition. The branch is a data-format
difference, not a gameplay one.

**Notes** — **the entire body is disabled.** An unconditional early return precedes it, with
a comment noting the intent to re-enable it before release, which never happened. So the
shipped games do not autosave through this manager at all; autosaves in those games come
from script calls at story checkpoints. A rebuild should treat this file as a *design that
was not switched on* — the readiness protocol below is live and used, but the timer is not.

## the readiness protocol

**Contract** — any subsystem that must not be interrupted by a save brackets its critical
region with an increment and a decrement of the not-ready counter. `ready_for_autosave` is
true exactly when the count is zero.

```text
FUNCTION inc_not_ready()      not_ready_count = not_ready_count + 1
FUNCTION dec_not_ready()      REQUIRE not_ready_count > 0
                              not_ready_count = not_ready_count - 1
FUNCTION ready_for_autosave() -> bool   RETURN not_ready_count == 0
```

**Notes** — this is the part of the file that is actually used, and it outlives the timer.
It is the general answer to "is the world in a saveable state", and anything that saves —
including the player's own save — can ask it.

## `on_game_loaded`

**Contract** — resets the last-save timestamp to now, so that loading a game does not
immediately trigger an autosave from an interval that elapsed before the load.

## `delay_autosave` and `update_autosave_time`

**Contract** — the two ways the timestamp moves: forward by the delay interval when a save
was refused, and to now when one was taken.
