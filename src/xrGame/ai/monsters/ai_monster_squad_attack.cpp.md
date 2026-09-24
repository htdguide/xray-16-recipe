# src/xrGame/ai/monsters/ai_monster_squad_attack.cpp

> Spreads a pack around a shared enemy: every member gets a distinct approach bearing, so a pack surrounds its prey instead of queueing behind it.

**Needs** — [`ai_monster_squad.h`](ai_monster_squad.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`monster_home.h`](monster_home.h.md) · [`entity_alive.h`](../../entity_alive.h.md)
**Used by** — [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md)
**Tier floor** — T3: a sort and bearing arithmetic over a handful of members

## Purpose

The visible signature of a pack in this game is the encirclement: dogs fan out and come at
the player from several sides at once. This file is where that happens. It groups members
by the enemy they each reported, and for each group assigns every member an *approach
bearing* — a direction, measured from the enemy, along which that member should close.

It also assigns each member a **pack index**, which is how the individual creature brains
stagger themselves: index one leads the charge, higher indices hang back or circle.

Two entirely different bearing algorithms live here. One is used; the other is not, and the
call site that chose between them is commented out. Both are documented because the
disabled one is the better design and a rebuild may want it.

## `coordinate_attack`

**Contract** — groups members whose reported goal is *attack* by the enemy they named, then
assigns bearings within each group. Clears and reuses scratch lists. No allocation beyond
those.

```text
FUNCTION coordinate_attack()
  groups = empty map<enemy, list<member>>
  FOR EACH (member, goal) IN goals
    IF goal.type = attack_enemy THEN groups[goal.entity].append(member)

  FOR EACH (enemy, members) IN groups
    assign_bearings_by_fan(members, enemy)
```

**Invariants** — a member with an attack goal must name a live, undestroyed enemy; the
coordination pass in [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) has already cleared
goals whose subject is gone, and this routine asserts the rest.

## `assign_bearings_by_fan`

**Contract** — the algorithm actually in use. Gives each member a bearing measured as an
offset from the *closest* member's bearing to the enemy, fanning alternately left and right.
Destroys the member list it is given. Writes one attack command per member.

```text
FUNCTION assign_bearings_by_fan(members, enemy)
  sort members by distance to the enemy, farthest first
  step = 2PI / count(members)          # equal shares of a full circle

  nearest = members.pop_last()         # the closest member
  nearest.offset = 0                   # it keeps the bearing it already has
  farthest = members.pop_first()       # the most distant member, if any
  farthest.offset = PI                 # goes to the far side

  next_right = step ; next_left = step
  WHILE members remain
    m = members.pop_last()             # next-closest first

    # which side of the nearest member is this one already on?
    bearing_of_nearest = heading from nearest's position to the enemy
    bearing_of_m       = heading from m's position to the enemy
    side = sign of normalized(bearing_of_m - bearing_of_nearest)

    # prefer that side; fall back to the other once this side has filled
    # a half-circle (within three degrees)
    IF side is right AND next_right < PI THEN take right
    ELSE IF side is left AND next_left < PI THEN take left
    ELSE take the other side

    m.offset = ± the chosen side's next value ; advance that side by step

  base = heading from the nearest member's position to the enemy
  FOR EACH m
    issue attack command to m: enemy, direction = base rotated by m.offset,
                               pitch taken from base
```

**Invariants** — the bearings are offsets from the nearest member's *existing* bearing, not
from a world axis. That is what makes the fan stable: the member already closest keeps
coming straight on and everybody else arranges around it, rather than the whole pack
re-orienting whenever the enemy moves.

The direction handed to the member is a **direction to approach from**, expressed as a full
heading-and-pitch vector; the pitch is copied from the nearest member's bearing for every
member, so all members approach in the same vertical plane.

**Notes** — the "already on this side" test is what keeps a member from being told to cross
in front of the enemy to reach its slot. It is a greedy assignment and is order-dependent
through the distance sort, which is deliberate: the closest members get the smallest
offsets and therefore the shortest detours.

The three-degree tolerance on the half-circle test (a sixtieth of pi) exists because the
step is a division and the accumulated sum overshoots pi by a rounding error on the last
slot. Without it the final member is pushed to the wrong side.

The scratch list of assigned bearings is a member of the pack rather than a local, which is
an allocation-avoidance habit and decides nothing.

## `assign_bearings_around_home`

**Contract** — the *disabled* alternative. Gives each member a bearing computed from its
pack index and the direction from the pack's **home point** to the enemy, so the members
divide a circle centred on the enemy in index order. Independent of where the members
currently are.

```text
FUNCTION bearing_for(member, enemy)
  home_to_enemy = enemy.position - member.home.point
  IF that is degenerate (enemy standing on the home point) THEN
    RETURN direction from the enemy to the member       # any stable direction
  IF the member is also on the enemy THEN FAIL          # asserted, not handled

  heading, pitch = polar of home_to_enemy
  heading = heading + 2PI * member.pack_index / living_member_count
  RETURN unit vector from (heading, pitch)
```

**Notes** — this is a cleaner design than the fan: the assignment is a pure function of the
index, so it is stable frame to frame and does not depend on a sort. It was written, wired
up behind a test for "is this a pack of creatures rather than humans", and then the test was
commented out in favour of calling the fan unconditionally. Nothing records why. The most
likely reason is that it anchors on the home point, which is meaningless for a pack that has
been drawn far from its territory.

A member whose pack index has never been assigned reads as an unset value and is treated as
index zero, with a comment in the original questioning whether that is right. It is not
recoverable whether it is.

## `assign_pack_indices` (and the rat variant)

**Contract** — sorts every living member by distance to a given enemy and numbers them from
**one**, closest first. The general variant additionally sets that enemy as each member's
current enemy; the rat variant only numbers.

```text
FUNCTION assign_pack_indices(enemy, also_set_enemy)
  members = every living member
  sort by distance to enemy, farthest first
  i = 0
  WHILE members remain
    i = i + 1
    m = members.pop_last()             # closest first
    m.pack_index = i
    IF also_set_enemy THEN m.set_enemy(enemy)
```

**Invariants** — indices start at one, not zero, and are dense over the living members. A
member that dies leaves a gap until the next assignment, which is why the caller re-runs
this whenever the pack's composition or target changes rather than maintaining it
incrementally.

**Notes** — the two variants differ only in whether the enemy is also assigned, and the rat
variant exists because rats are handled as a swarm whose members must not each independently
acquire the enemy. Both build a single-entry map keyed by the enemy before iterating it,
which is pure overhead — there is only ever one group — and a rebuild drops it.

The general variant hard-casts each member to the creature base, so a pack containing a
non-creature entity would fail. Nothing constructs such a pack.
