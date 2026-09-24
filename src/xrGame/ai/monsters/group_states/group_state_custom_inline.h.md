# src/xrGame/ai/monsters/group_states/group_state_custom_inline.h

> Stand still and play the idle clip the brain asked for by number, with a sound chosen from that
> number.

**Needs** — [`group_state_custom.h`](group_state_custom.h.md) · [`../states/state_custom_action.h`](../states/state_custom_action.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_custom.h`](group_state_custom.h.md)
**Tier floor** — T3: a delegation with a three-way sound lookup

## Purpose

The bridge between the pack states' *decision* machinery and the creature's *numbered animation
vocabulary* (see [`../dog/dog.cpp`](../dog/dog.cpp.md)). The pack states decide "play clip 6" by
writing the number into the creature and selecting this state; this state makes the clip actually
play and holds the creature still while it does.

It exists as a state rather than as a call because a playing clip has to *own* the brain — the
selector above must not re-decide while a sniff or a howl is mid-animation, and the completion of
the clip is what advances the sequence. Wrapping it in a state is what gives it a lifetime the
brain respects.

## State

`Stateless.` It owns one substate — the generic "hold this action until told otherwise" leaf — and
reads everything else from the creature.

## `CStateCustomGroup`

**Contract** — each tick: select the single substate (a no-op once selected), ask the creature to
start the requested clip, and run the substate. Finished when the creature reports the clip has
ended. Never blocks, never allocates.

```text
FUNCTION execute()
  select(hold_action_substate)
  creature.start_requested_animation()    # captures the animation channel; no-op if already held
  substate.execute()
  previous = active

FUNCTION is_finished() -> bool
  RETURN creature.animation_finished_flag

FUNCTION setup_substate()
  action = stand_idle
  time_out = none                          # the clip's own length ends the state
  sound = clip number 5 -> the stealth sound       # the howl
          clip number 6 -> the threat sound        # the growl
          otherwise     -> the idle sound
  sound_delay = the section's eating-sound delay
```

**Notes** — the sound is selected from the *clip number*, which is the only place in the engine
where the numbered vocabulary is interpreted outside the creature class. Two numbers are special —
the howl and the growl each get their own sound category — and everything else is an idle noise. A
rebuild that names the vocabulary members should name these two, because they are the two the rest
of the system knows about.

The timeout is deliberately disabled: the state's length is the clip's length, and the clip
reports its own end through the creature's animation callback. That is the same inversion
described in [`../dog/dog.cpp`](../dog/dog.cpp.md), and it is why the brain must not impose a
duration here.

The sound *delay* is read from the eating-sound delay regardless of which sound was chosen. That
looks like a copy from a neighbouring state rather than a decision, and nothing recoverable
explains it.

Requesting the clip every tick rather than once on entry is safe because the creature's own
start-animation routine refuses when the animation channel is already held. The state is therefore
idempotent per tick, which matters because it is re-entered from several different selectors.
