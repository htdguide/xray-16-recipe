# src/xrGame/ai/monsters/flesh/flesh.cpp

> The flesh: a near-pure data creature — an animation table, a narrower eye arc, and one geometry
> routine that survives from a trampling attack the shipped creature no longer performs.

**Needs** — [`flesh.h`](flesh.h.md) · [`flesh_state_manager.h`](flesh_state_manager.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [Seam: Configuration (ltx)](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`flesh.h`](flesh.h.md)
**Tier floor** — T3: an animation table plus one closed-form geometric predicate

## Purpose

The flesh is the chapter's clearest example of the pattern the whole chapter rests on: **a
creature is its data**. Everything it does — wandering, fleeing, attacking, eating, dragging
corpses — comes from the shared base creature and the generic states. What this file supplies is
the animation vocabulary that makes those behaviours look like a flesh, one widened-vision
constant, one animation-flourish hook, and one geometry helper.

A rebuilder should read this file as the template for every other data-only creature in the
chapter, and then read only the *differences* in the creatures that have them.

## State

The creature adds exactly one field to the base creature, set in its constructor and never
changed:

```text
RECORD Flesh EXTENDS BaseMonster, ControlledEntity
  eye_shift_yaw : real   # the vision cone is rotated by 30 degrees off the body axis
```

**Notes** — the flesh's eyes are on the sides of its head, and the base creature's vision test
takes a yaw offset precisely so that a creature can be given side-set eyes. Thirty degrees is
authored in code, not in the creature's section, which makes it the one piece of flesh perception
a modder cannot retune.

## `Load`

**Contract** — declare the creature's animation vocabulary: the damaged-variant substitutions, the
walk-to-run acceleration chains, the clip registrations with their velocity profiles and postures,
the posture transition graph, and the mapping from abstract actions to clips. Every clip name is a
prefix into the shipped animation bank and is frozen against the game data.

```text
FUNCTION load(section)
  substitute(when damaged: run -> run_damaged, walk -> walk_damaged)
  chain(walk -> run), chain(walk_damaged -> run_damaged)

  register the vocabulary: idle, two turns, lie idle, sleep (reusing lie idle),
     walk and run in healthy and damaged variants, a forward attack, a
     back attack, corpse inspection (reusing the eat clip's second variation),
     eat, die, the two posture transitions, a crawl used as the stealth gait,
     a drag gait, a scared stance and a threat display

  transition(stand -> lie)
  transition(lie -> stand, skip when aggressive)

  link every abstract action to its clip;
     rest and both seated actions all resolve to lying idle,
     and "look around" resolves to the scared stance
```

**Notes** — two links carry character rather than mechanism. The flesh has no seated posture at
all, so *sit* is mapped onto *lie*; and its "look around" is the **scared** stance, not a neutral
one, which is why a flesh that has heard something reads as nervous rather than curious. Those are
one-line authoring decisions with a visible effect, and they are exactly the kind of thing a
rebuild loses by generating the action table mechanically.

Sleep and lying idle are the same clip. The flesh does not sleep differently from resting; the
distinction exists in the state machine and not in the data.

## `CheckSpecParams`

**Contract** — the hook a state uses to request a one-off animation flourish. Four flags are
honoured: *drag a corpse* and *threaten* replace the current clip; *inspect a corpse* and *attack
from behind* run their clips as one-shot sequences that own the body until they finish.

**Notes** — the split between *setting* a clip and *running a sequence* is the same one described
for the dog: a display may be overwritten by the next tick's action, an act that has consequences
must complete. A back attack lands damage at a moment inside the clip, so it must be a sequence.

## `net_Spawn`

**Contract** — forwards to the base creature and adds nothing. Present only so the class can be
extended later without touching call sites; a rebuild can omit it.

## `ConeSphereIntersection`

**Contract** — a pure, closed-form predicate: does an unbounded cone, given by apex, axis and
half-angle, intersect a sphere given by centre and radius? No allocation, no side effects, no
dependence on world state.

```text
FUNCTION cone_intersects_sphere(apex, half_angle, axis, centre, radius) -> bool
  # shift the sphere's centre along the axis by radius / sin(half_angle):
  # this converts "cone versus sphere" into "point versus a widened cone"
  d = (centre - apex) + axis * (radius / sin(half_angle))

  IF d . axis > 0 AND (d . axis)^2 >= |d|^2 * cos(half_angle)^2
    # inside the widened cone; now reject the case where the sphere lies
    # entirely behind the apex, in the mirrored cone
    e = (apex - centre) . axis
    IF e > 0 AND e^2 >= |centre - apex|^2 * sin(half_angle)^2
      RETURN |centre - apex|^2 <= radius^2      # apex inside the sphere
    RETURN true
  RETURN false
```

**Notes** — the source states its purpose: deciding whether a flesh's *hoof* sweeps through a
rat's bounding sphere. Nothing in the shipped code calls it. The trampling attack it was written
for either never shipped or was moved into the generic melee path, and the routine survives as a
private member of the creature class — which is the wrong home for a piece of pure geometry in
any case.

A rebuild has two honest options: drop it, or lift it into the math layer where a cone/sphere test
belongs. Keeping it as a creature method reproduces a mistake. The algorithm itself is a standard
and correct treatment — the shift of the sphere centre along the axis is the trick that makes the
test closed-form, and the second branch is the well-known correction for the mirrored cone behind
the apex.
