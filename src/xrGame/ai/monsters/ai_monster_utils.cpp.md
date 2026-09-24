# src/xrGame/ai/monsters/ai_monster_utils.cpp

> Two questions asked constantly by creature code: is this entity actually standing where the navigation mesh thinks it is, and where in the world is a named bone.

**Needs** — [`ai_monster_utils.h`](ai_monster_utils.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location_impl.h`](../../../xrAICore/Navigation/ai_object_location_impl.h.md) · [`ai_space.h`](../../ai_space.h.md) · [`Include/xrRender/Kinematics.h`](../../../Include/xrRender/Kinematics.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — [`ai_monster_utils.h`](ai_monster_utils.h.md)
**Tier floor** — T2: a navigation-mesh containment test and a bone transform composition

## Purpose

Implements the four non-inline helpers of [`ai_monster_utils.h`](ai_monster_utils.h.md).
Two of them matter.

## `object_position_valid`

**Contract** — is this entity's world position genuinely inside the navigation mesh cell it
claims to occupy? Three conditions must all hold: the claimed vertex identifier is a real
vertex, the world position is within the mesh's bounds at all, and the position lies inside
that specific vertex's cell.

**Invariants** — an entity can be physically pushed off its cell — knocked back, thrown by
an anomaly, standing on a crate — while still carrying the last vertex it legitimately
occupied. Every piece of creature code that is about to *reason* about a position with the
pathfinder must ask this first, because the pathfinder's answers are only meaningful for a
position on the mesh.

## `get_valid_position`

**Contract** — given an entity and a position, returns that position unchanged if the
entity is properly on its mesh cell, and otherwise returns the **centre of the cell** it
claims. The fallback is always a legal navigation position.

**Notes** — the substitution is what keeps a creature's plan sane while it is momentarily
off-mesh: it plans from the nearest legal place instead of from nowhere. The cost is that
the creature's plan briefly disagrees with where it visibly is, which is the lesser
problem.

## `get_bone_position`

**Contract** — the world position of a bone, by name. Resolves the name to a bone index in
the model, reads that bone's current animated transform, composes it with the object's own
transform, and returns the translation. Does a name lookup every call.

**Notes** — the name-to-index resolution is per call, which is wasteful and is why callers
that need a bone every frame cache the index themselves. A rebuild resolves once at spawn.

## `get_head_position`

**Contract** — the world position of an object's head. Uses the object's own declared head
bone if it is a creature; otherwise falls back to the human skeleton's head bone name.

**Notes** — the fallback name is the human rig's, which is correct for the player and for
humans and would be wrong for any non-creature with a different skeleton. Nothing else asks.
Creatures declare their own head bone because their skeletons are all different, and that
declaration is the only reason this function is not just a name constant.
