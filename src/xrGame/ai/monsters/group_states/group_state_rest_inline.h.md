# src/xrGame/ai/monsters/group_states/group_state_rest_inline.h

> The idling brain: obey a smart terrain if one is calling, stay inside your permitted region and
> your territory, and otherwise run a sleep/wake cycle punctuated by flavour animations.

**Needs** — [`group_state_rest.h`](group_state_rest.h.md) · [`group_state_rest_idle.h`](group_state_rest_idle.h.md) · [`group_state_custom.h`](group_state_custom.h.md) · [`../states/monster_state_rest_sleep.h`](../states/monster_state_rest_sleep.h.md) · [`../states/state_move_to_restrictor.h`](../states/state_move_to_restrictor.h.md) · [`../states/monster_state_home_point_rest.h`](../states/monster_state_home_point_rest.h.md) · [`../states/monster_state_smart_terrain_task.h`](../states/monster_state_smart_terrain_task.h.md) · [`../anomaly_detector.h`](../anomaly_detector.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_rest.h`](group_state_rest.h.md)
**Tier floor** — T3: a four-level priority ladder plus a clip-sequence state machine

## Purpose

This is the state the player sees most, because most creatures on a level are idling most of the
time. It has two halves and they are worth separating in the mind:

**A priority ladder** over four claims on the creature's time — a smart terrain's job, a movement
restriction it is outside of, its own territory, and finally its own devices. Each rung uses the
same latch idiom: *if I was doing this, am I done; otherwise, should I start*.

**A sleep/wake cycle** at the bottom of the ladder, expressed through the numbered animation
vocabulary. This is the half that makes a pack of dogs look alive: they wander, sit, scratch,
stand, lie down, sleep, and get up, each transition being a specific clip requested by number.

## State

```text
RECORD GroupRestState
  time_for_life  : int   # when this waking period ends and sleep becomes possible
  time_for_sleep : int   # when the current sleep ends
```

Both are set from the creature's authored timeouts multiplied by a random factor:

- waking period = `min_life_time + random(0..9) * min_life_time` — so between one and ten times
  the authored figure
- sleep period = `min_sleep_time + random(0..4) * min_sleep_time` — between one and five times

**The random multiplier is the load-bearing part.** A pack whose members all share a section would
otherwise sleep and wake in lockstep. Multiplying rather than adding jitter means the spread scales
with the authored value, so a creature authored to sleep briefly has brief, tightly spread naps
and one authored to sleep long has long, widely spread ones.

## `initialize` / `finalize` / `critical_finalize`

**Contract** — on entry, clear the sleep deadline, draw a fresh waking period, and **arm the
anomaly detector**. On either exit, disarm it.

**Notes** — the anomaly detector is a separate component that steers a creature away from the
level's hazard volumes. Arming it only while idling is deliberate and visible in play: a creature
wandering its territory avoids anomalies, and a creature charging an enemy runs straight through
them. That is one of the small rules that makes anomalies a usable tactic for the player.

## `execute`

**Contract** — walk the ladder, select one substate, run it, record it. Several branches return
early after running their substate rather than falling through to the common tail — the common
tail does the same thing, so the early returns are redundant, but they are what the original does
and they make the sleep-cycle branches self-contained.

```text
FUNCTION execute()
  # rung 1: a smart terrain has offered this creature a job
  IF (was doing the job AND not finished) OR (job will accept)
    select smart_terrain_task
  ELSE
    # rung 2: we are outside a movement restriction we must be inside
    IF (was moving to the restriction AND not finished) OR (it will accept)
      select move_to_restrictor
    ELSE
      # rung 3: we are outside our own territory
      IF (was going home AND not finished) OR (going home will accept)
        select go_home
      ELSE
        run the sleep/wake cycle        # rung 4
  active.execute(); previous = active
```

**Notes on the ladder** — the order encodes who owns the creature's time, from most external to
most internal: the level's authored places first, then the level's authored boundaries, then the
creature's own territory, then the creature itself. A rebuild that reorders these produces
creatures that ignore smart terrains, which is the single most visible failure mode in the whole
chapter because smart terrains are how the world appears populated.

## The sleep/wake cycle

The bottom rung, expressed entirely through the numbered animation vocabulary. The clip numbers
are the dog's (see [`../dog/dog.cpp`](../dog/dog.cpp.md)) and their *adjacency* is part of the
contract.

```text
# --- waking up, driven by where in the sequence the creature currently is
IF we are marked as sleeping
  IF the current clip is 8  (sit down)     -> request 13 (lie down)
  IF the current clip is 14 (rise from lying) -> request 12 (stand up)
  IF the current clip is 12 (stand up)     -> request 7  (shake itself), and stop being asleep
  IF a clip request is pending
    select the flavour-animation state; clear the request; run it; RETURN

# --- staying asleep
IF the sleep deadline has not passed AND we are asleep AND the current clip is 13
  select sleep; run it; RETURN

# --- ending a sleep
IF the previous rung was sleep
  IF the sleep deadline has not passed     -> stay asleep (fall through, no new selection)
  ELSE
    draw a fresh waking period
    request 14 (rise from lying); select the flavour-animation state; run it; RETURN

# --- beginning a sleep
IF the waking period has expired AND we are inside the inner home ring
  request 8 (sit down); select the flavour-animation state
  mark ourselves as sleeping
  draw the sleep period
  run it; RETURN

# --- otherwise: advance an idle sequence, or idle
IF we are not asleep AND the previous rung was a flavour animation
   AND the current clip number is in 8..11
  request the next clip number; select the flavour-animation state; run it; RETURN

IF a clip request is pending  -> the flavour-animation state
ELSE                          -> idle in the territory
```

**Notes** — three things here are worth carrying into a rebuild even if the clip numbers are
replaced by names.

**Falling asleep is a sequence, not a state change.** Going to sleep is clip 8 (sit down) then 13
(lie down); waking is 14 (rise from lying), 12 (stand up), 7 (shake itself). Each is a separate
activation of the flavour-animation state, and the transitions are driven by *reading back which
clip the creature last played*. The creature's animation machine is therefore the sequencer, and
this code is a table of successors. A rebuild with an explicit sequence is clearer and must
reproduce the same five clips in the same order.

**Sleep requires being deep inside the territory.** A creature only lies down when it is within the
inner home ring; out on the edge it keeps wandering however tired it is. That is what keeps
sleeping creatures where the level designer put their home point.

**The seated idle sequence is a run of consecutive numbers.** Clips 8 through 11 — sit down, sit,
scratch while seated, look around while seated — are advanced by *incrementing the number*, which
is why the vocabulary's ordering is a contract and not an arbitrary tagging. The run stops at 11
and the creature returns to ordinary idling.

The fall-through case when the sleep deadline has not passed selects nothing new and lets the
previously active rung continue — the only place in the file where the ladder deliberately
declines to choose.
