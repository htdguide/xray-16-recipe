# src/xrGame/ai/monsters/control_animation_base_load.cpp

> Builds the four tables that turn a creature's abstract actions into clips: the motion table, the transition list, the action map and the conditional substitutions.

**Needs** — [`control_animation_base.h`](control_animation_base.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: table construction at load time, once per creature

## Purpose

Every concrete creature's load routine is, for the most part, a long run of calls into
this file: here is the clip for standing idle, here is the clip for running, here is the
bridge from standing to sitting, this action means that motion, and while wounded play
this instead of that. The tables built here are read every frame by
[`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) and never
change afterwards.

The file is a separate file for a good reason: it is the boundary where a creature's
authored animation set enters the engine, and it is the only place that knows the
model-probing convention.

## State

Writes into the animation base's tables; owns nothing of its own. See
[`control_animation_base.h`](control_animation_base.h.md) for the records.

## `AddAnim`

**Contract** — register one motion. Takes the motion identifier, the clip-name *prefix*,
a variant family selector, the ground speed the clip was authored at, the posture the clip
belongs to, and up to four directional hit-reaction clip names. Answers whether the motion
was registered. Allocates one table entry; the entry is owned by the animation base for
the creature's lifetime.

When the caller declares the motion **optional**, the model is probed first and a model
that has neither the prefixed-with-zero clip nor the bare prefix declines silently. When
the motion is required — the default — a missing clip is not detected here and surfaces
later as a failed lookup. The distinction exists because several creature families share
one load routine and differ by a handful of clips.

The hit-reaction clip names are each probed against the model and only the ones that exist
are kept, so a creature may declare all four and get whichever its model supplies.

```text
FUNCTION add_anim(motion, prefix, family, velocity, posture, effects, required) -> bool
  IF NOT required AND model has neither (prefix + "0") nor prefix THEN
    RETURN false
  entry <- new AnimationItem
  entry.target_name <- prefix
  entry.spec_id     <- family        # -1 means any variant is acceptable
  entry.velocity    <- velocity
  entry.posture     <- posture
  entry.count       <- 0             # filled later by probing the model
  FOR EACH side IN { front, back, left, right }
    IF effects[side] is non-empty AND the model has that effect clip THEN
      entry.effects[side] <- effects[side]
  anim_storage[motion] <- entry
  RETURN true
```

**Notes** — the variant count starts at zero and is discovered by a separate pass that
probes the model for successive indices. That is why registration cannot fail on a missing
required clip: nothing has looked yet.

## `AddTransition`

**Contract** — register a bridge clip to play when the creature moves from one animation
state to another. Four spellings: motion-to-motion, motion-to-posture, posture-to-motion,
posture-to-posture. Each records whether the bridge may itself be followed by another
bridge (*chained*), and whether it should be skipped when the creature is in an aggressive
hurry.

The four spellings are one operation with a flag on each end saying whether that end names
a posture or a motion. A rebuild should write one function taking a two-case value on each
side; the four overloads are a C++ convenience with no meaning of their own.

**Notes** — the *posture* spellings are the useful ones. A creature that must stand up
before it can walk declares one transition from the sitting posture to the standing
posture, and every sitting clip is then covered. The motion-to-motion spelling exists for
the exceptions.

## `LinkAction`

**Contract** — map an abstract action to a motion. The second spelling adds a left-turn
clip, a right-turn clip, and the heading error past which the turn clip replaces the
straight one. An action maps to exactly one entry; registering twice keeps the first.

**Notes** — turn clips are attached to the *action*, not to the motion, so the same walk
clip can turn differently depending on why the creature is walking. In practice almost no
creature uses this and the plain spelling dominates.

## `AddReplacedAnim`

**Contract** — register a conditional substitution: while a named boolean on the creature
is true, play one motion in place of another. The condition is held as a live reference to
the creature's own field, so the substitution follows the creature's state with no
per-frame plumbing.

**Notes** — the live reference is the incidental part; what a rebuild needs is a
*predicate* evaluated at selection time. Every shipped use points at the creature's
"wounded" flag, which is how a damaged creature limps without a second copy of every
action mapping.
