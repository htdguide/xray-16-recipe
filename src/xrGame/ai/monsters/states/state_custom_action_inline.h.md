# src/xrGame/ai/monsters/states/state_custom_action_inline.h

> The generic action leaf: play what you were told, for as long as you were told, making the noise
> you were told to make.

**Needs** — [`state_custom_action.h`](state_custom_action.h.md) · [`state_data.h`](state_data.h.md)
**Used by** — [`state_custom_action.h`](state_custom_action.h.md)
**Tier floor** — T3: three requests and a timer comparison

## Purpose

Thirty lines that remove perhaps a thousand. Every composite in this directory needs leaves that
do nothing but hold a pose — sniff a corpse, cower, look around, rest at a job's spot — and every
one of those differs only in four values. This is that leaf, once.

The idea a rebuild must carry across is not the class but the **arrangement**: a leaf owns a
parameter block, the composite above it writes that block whenever it selects the leaf, and the
leaf reads it. Combined with the reselection step, that lets one leaf object serve as several
behaviourally distinct steps within the same composite at different times.

## State

```text
RECORD ActionLeafParameters          # the shape is defined in state_data.h
  action      : action identifier    # stand idle, rest, look around, ...
  spec_params : int (bit flags)      # animation variant: scared, threatening, ...
  time_out    : int (milliseconds)   # 0 means "never finish"
  sound_type  : optional<int>        # absent means "play nothing"
  sound_delay : optional<int>        # absent means "no repeat throttle"
```

**Invariant** — the block is owned by the leaf and its address is handed to the state base at
construction. The base's parameter-filling operation copies a caller-supplied record over it
wholesale, by size. That makes the arrangement **size-checked only by the caller**: a composite
that writes the wrong record type, or the right type at the wrong size, corrupts the block
silently. One such mismatch exists in this directory and is documented in
[`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md).

A rebuild that gives each leaf a typed parameter setter removes that whole class of defect and
loses nothing.

## `execute`

**Contract** — set the animation layer's current action and its variant flags from the block, and
play the voice if one is named — throttled by the repeat delay if one is given, unthrottled
otherwise. Runs every update, restating the same values.

```text
FUNCTION execute()
  animation.action      = parameters.action
  animation.spec_params = parameters.spec_params
  IF parameters.sound_type EXISTS
    IF parameters.sound_delay EXISTS  voice.play(sound_type, repeat_delay = sound_delay)
    ELSE                              voice.play(sound_type)
```

**Notes** — the action is written **directly onto the animation component**, not through the
creature's action-request operation that every other leaf in this directory uses. The difference is
that the creature's operation also updates its own record of what it is doing, which the debug
overlay and a handful of scripted queries read. A rebuild that routes this through the same path as
everything else changes nothing the player sees and makes the leaf consistent with its siblings;
reproducing the original exactly means this one leaf leaves the creature's own action record
untouched.

An absent voice is a real configuration, not an omission — the fear behaviours use it to keep a
frightened creature silent while still giving the leaf a repeat delay it never uses.

## `check_completion`

**Contract** — finished once the stored timeout has elapsed since entry. A timeout of zero means
the leaf never finishes on its own.

**Notes** — zero as "never" is the convention throughout this directory and it is what makes the
absorbing leaves absorbing: the cowering step, the mill-around step, the wait-at-the-job step are
all this leaf with a zero timeout. A rebuild that uses an absent value instead must be sure the
default is "never" and not "immediately", because the parameter record's default is zero and
several composites rely on not setting the field at all.
