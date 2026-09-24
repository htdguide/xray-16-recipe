# src/xrGame/ai/monsters/control_sequencer.cpp

> Plays a list of clips back to back on the whole body, one per animation-end event, and reports when the list runs out.

**Needs** — [`control_sequencer.h`](control_sequencer.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_animation.h`](control_animation.h.md)
**Used by** — reached through its declarations in [`control_sequencer.h`](control_sequencer.h.md); callers name that, not this file.
**Tier floor** — T2: seizes the body for the duration of a clip list

## Purpose

The simplest of the layer-two animation clients. It exists so that "play these three clips
in order and tell me when you are done" does not have to be written again in every creature
that needs it — checking a corpse, an authored idle flourish, a scripted gesture.

Unlike the abilities, it holds the body but drives nothing: path, movement and direction are
all stopped for the duration, and only the animation advances.

## State

```text
RECORD SequencerPayload
  motions : list<Motion>

RECORD Sequencer
  index : int      # which entry of the list is playing
```

## `on_capture`

**Contract** — seize all four body resources, subscribe to animation end, and stop the path,
the movement and the turn. Note that this happens on *capture*, not on activation: the
sequencer takes the body as soon as someone claims it, so that the claimant can fill in the
clip list while the creature is already standing still.

**Notes** — every other custom element in this slice does its seizing in `activate`. The
difference matters: a caller of the sequencer builds its list in two or more calls between
capture and activation, and the creature must not be moving during those calls.

## `activate`

**Contract** — reset the index to the start of the list and play the first clip.

## `on_event`

**Contract** — on animation end, advance to the next clip if there is one, otherwise raise
the sequence-end event.

## `play_selected`

**Contract** — write the current clip into the animation payload and mark it stale, which is
how every element in this chapter asks the mixer to start something.

## `reset_data` / `on_release` / `check_start_conditions`

**Contract** — `reset_data` empties the clip list, and because the manager resets a payload
on capture, each capture starts from an empty list. `on_release` hands the body back and
unsubscribes. `check_start_conditions` refuses if the sequencer is already running or if any
body resource is held by another ability.
