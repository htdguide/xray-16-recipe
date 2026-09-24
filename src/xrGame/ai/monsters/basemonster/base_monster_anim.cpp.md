# src/xrGame/ai/monsters/basemonster/base_monster_anim.cpp

> The animation-selection entry point, which a creature answers by handing the question straight to its animation control channel.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`control_animation_base.h`](../control_animation_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a delegation

## Purpose

The entity layer above asks every animated entity to select its animation each frame, handing
it a view direction, a movement direction and a speed. A creature ignores all three: its
animation has already been decided by its state machine and written into the animation
control channel, and all this does is tell that channel to apply what it holds.

The file exists as a file only because the entity layer's virtual question has to be answered
somewhere. A rebuild that lets a creature opt out of the question deletes it.

## `select_animation`

**Contract** — advances the animation control channel's frame pass. Ignores its three
arguments entirely.

**Notes** — the ignored arguments are the whole point and the whole content of the file: a
human's animation is chosen *from* its view and movement vectors, and a creature's is not.
A creature's animation is a consequence of the abstract action the state machine chose,
resolved through the creature's animation table. See
[`ai_monster_defs.h`](../ai_monster_defs.h.md).
