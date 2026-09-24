# src/xrGame/ai/monsters/control_melee_jump.cpp

> The melee jump: a standing creature with its enemy behind it and within arm's reach spins to face it in one clip.

**Needs** — [`control_melee_jump.h`](control_melee_jump.h.md) · [`control_manager.h`](control_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — reached through its declarations in [`control_melee_jump.h`](control_melee_jump.h.md); callers name that, not this file.
**Tier floor** — T2: seizes the body for the duration of one clip

## Purpose

The smallest ability in the chapter and the clearest illustration of the pattern. It has no
path, no movement, no physics: it seizes the body, aims the heading at the enemy, plays one
of two clips depending on which way the turn goes, and ends when the clip does. Everything
it does is a subset of what the rotation jump does.

It exists as a separate ability rather than a mode of the rotation jump because its start
conditions are different in kind: the rotation jump is for a creature at speed, this is for
one standing in contact.

## State

```text
RECORD MeleeJump
  time_next_melee_jump : int (ms)

RECORD MeleeJumpPayload
  anim_ls, anim_rs : Motion    # the turn clip for each side
```

Three constants, in this file rather than in data: a cooldown drawn uniformly from 500 to
1000 milliseconds, a facing check of 165 degrees, and a maximum distance to the enemy of
four world units.

## `check_start_conditions`

**Contract** — the ability is not running; no other ability holds the body; an enemy
exists; the cooldown has expired; the enemy is at least 165 degrees off the creature's
facing; and the enemy is within four world units.

**Notes** — 165 degrees is a narrower window than the rotation jump's 150: this clip is a
near-complete about-face and looks wrong used for a lesser turn. Four units is close
enough that the creature is effectively in contact.

## `activate`

**Contract** — seize the body, subscribe to animation end, stop the path and the movement,
then aim and play.

```text
FUNCTION activate()
  capture all four body resources; subscribe to animation_end
  stop the path; stop the movement

  target_yaw <- the heading that points at the enemy
  motion     <- IF the enemy is on my right THEN anim_rs ELSE anim_ls
  anim_time  <- duration of that clip

  heading target <- target_yaw
  heading rate   <- (heading error) / anim_time      # the turn and the clip end together
  linear dependency <- off
  REQUIRE the rate is non-zero

  start the clip
```

**Notes** — the rate-from-clip-duration rule appears here in its purest form, uncomplicated
by paths or physics. It is the chapter's standard way of marrying a body rotation to a clip
and a rebuild should factor it out.

## `on_release`

**Contract** — restore linear dependency, release the body, unsubscribe, and set the
cooldown half a second to a second ahead.

**Notes** — unlike every other ability, this one does not unlock the path channel, because
it never locked it. The copy-paste sibling abilities all do.

## `on_event`

**Contract** — animation end raises the melee-jump-end event.
