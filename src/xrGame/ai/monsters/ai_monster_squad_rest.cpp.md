# src/xrGame/ai/monsters/ai_monster_squad_rest.cpp

> Arranges an unalarmed pack around its leader: scattered ahead, behind and to the sides while it travels, gathered at its position while it rests.

**Needs** — [`ai_monster_squad.h`](ai_monster_squad.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md)
**Tier floor** — T3: four sorts and some vector arithmetic over a handful of members

## Purpose

The other half of the pack's coordination pass. When no member is attacking, the pack still
has to look like a pack: dogs travelling together spread out around the leader rather than
walking in its footprints, and dogs settling down gather near it.

Which of the two happens is decided by the **leader's own goal**, not by a pack-level state.
That is the load-bearing decision in this file: the pack has no mode of its own, it simply
mirrors whatever its leader decided to do.

## `coordinate_idle`

**Contract** — collects every member whose reported goal is *rest* or *travel*, then
dispatches on the leader's goal. Requires a live leader.

## follow formation (leader is travelling)

**Contract** — assigns each non-leader member a position to move to, distributed round-robin
across four zones — ahead of the leader, behind it, to its left, to its right — with each
member landing in the zone it is *already nearest to*. Issues a follow command naming the
leader.

```text
FUNCTION assign_follow_positions(members)
  # four zone centres, each 20 units from the leader along one of its axes
  ahead  = leader.position + leader.forward * 20
  behind = leader.position - leader.forward * 20
  right  = leader.position + leader.right   * 20
  left   = leader.position - leader.right   * 20

  # four candidate lists, each holding every member, each sorted by distance
  # to its own zone centre — farthest first, so "closest" is the list's tail
  FOR EACH zone: candidates[zone] = all members except the leader,
                                    sorted by distance to that zone's centre

  zone = ahead
  WHILE any candidates remain
    m = candidates[zone].pop_last()            # the member nearest this zone
    remove m from the other three lists
    target = zone centre + a random direction * a random radius in [10, 15]
    issue follow command to m: leader, target, leader's facing
    zone = next zone, cycling ahead -> behind -> left -> right
```

**Invariants** — a member is assigned exactly once; the removal from the other three lists
is what guarantees it. The round-robin over zones is what guarantees the pack spreads
rather than all piling into whichever zone is nearest the group's centre of mass.

**Notes** — the three distances are the whole shape of a travelling pack: the zone centres
sit 20 world units out along the leader's axes, and each member's actual target is jittered
within 10 to 15 units of its zone centre, in a **fully random direction including
vertically**. None of the three is derived. The vertical component of the jitter is almost
certainly unintended — the target is a position a ground creature must reach — but it is
harmless because the movement layer resolves the target onto the navigation mesh.

A pack of fewer than four followers leaves some zones empty, and one of five puts the fifth
back into the first zone. Both are correct outcomes of the cycle and neither is special-cased.

The leader itself is excluded from every list, so it keeps whatever goal it set for itself.
Nothing commands the leader, ever.

## rest formation (leader is resting)

**Contract** — issues every non-leader member a rest command naming the leader's exact
position and navigation vertex. No spread, no jitter.

**Notes** — the contrast with the travel case is deliberate and is the file's second
decision: a travelling pack must not bunch, because it would jam in corridors; a resting
pack must bunch, because a sleeping pack scattered over 20 units does not read as a pack.
The rest command carries a vertex as well as a position because the individual rest state
paths to a vertex, and resolving the leader's position to a vertex once here is cheaper and
more consistent than each member resolving it separately.
