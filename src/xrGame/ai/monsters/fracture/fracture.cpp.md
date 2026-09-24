# src/xrGame/ai/monsters/fracture/fracture.cpp

> The fracture: a data-only creature with a seated resting posture and no drag ability — its
> animation table is the whole of its identity.

**Needs** — [`fracture.h`](fracture.h.md) · [`fracture_state_manager.h`](fracture_state_manager.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [Seam: Configuration (ltx)](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`fracture.h`](fracture.h.md)
**Tier floor** — T3: an animation table and one flourish hook

## Purpose

The thinnest creature in the chapter. It adds *nothing* to the shared base creature except an
animation vocabulary and one flourish hook. It is not even controllable — unlike the dog and the
flesh, a controller cannot take a fracture over — so the class has a single base.

Read it as the floor of the chapter's pattern: this is the minimum a new creature must supply, and
everything beyond it in the other creature files is an explicit addition.

## State

`Stateless.` The creature adds no fields to its base.

## `Load`

**Contract** — declare the animation vocabulary: damaged-variant substitutions, the walk-to-run
acceleration chains, clip registrations with velocity profiles and postures, the posture
transition graph, and the mapping from abstract actions to clips. Clip names are prefixes into the
shipped animation bank and are frozen against the game data.

```text
FUNCTION load(section)
  substitute(when damaged: run -> run_damaged, walk -> walk_damaged)
  chain(walk -> run), chain(walk_damaged -> run_damaged)

  register: standing idle and a damaged standing idle, two turns,
     walk and run in healthy and damaged variants, an attack, a death clip,
     corpse inspection, a crawl used as the stealth gait,
     a seated idle with its two transition clips, a look-around, and a seated eat

  transition(stand -> sit), transition(sit -> stand)      # both mandatory

  link the abstract actions; note the three collapses:
     both seated and lying rest resolve to the seated idle,
     walking backward resolves to walking forward,
     and dragging resolves to standing idle
```

**Notes** — three of the action links are *collapses*, and each states something about the
creature. It has no lying posture, so lying resolves to sitting. It has no backward walk, so
walking backward plays the forward clip — a rebuild that reproduces this faithfully will show the
creature moonwalking in the rare states that request it, which is what the original does. And it
has no drag gait, because a fracture does not drag corpses; the link exists so that the action
table is total, not because the action is ever requested.

Unlike the dog and the flesh, **both posture transitions are mandatory**: a fracture always plays
the full stand-up animation, even when startled. That is a deliberate contrast — it is a slow,
heavy creature, and the two "skip when aggressive" flags that make the dog snap upright are absent
here.

It eats **seated**, which is the only creature in the chapter that does; the eat action resolves to
a seated clip, and the state machine's eat sequence therefore drives it through the posture
transition first.

## `CheckSpecParams`

**Contract** — the flourish hook. Two flags are honoured: *inspect a corpse* runs its clip as a
one-shot sequence that owns the body until it finishes; *stand scared* replaces the current clip
with the look-around and **returns immediately**, so no further flag in the same request is
examined.

**Notes** — the early return after the scared flag is the one ordering decision in the file, and it
matters: the two flags can be requested together, and the scared display wins. A rebuild that
evaluates flags independently gives a different result.
