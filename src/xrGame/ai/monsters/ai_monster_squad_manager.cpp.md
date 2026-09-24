# src/xrGame/ai/monsters/ai_monster_squad_manager.cpp

> Owns every creature pack, addressed by the team/squad/group triple each creature already carries, and runs one pack's coordination when its leader thinks.

**Needs** — [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md) · [`ai_monster_squad.h`](ai_monster_squad.h.md)
**Used by** — [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md)
**Tier floor** — T3: a three-level nested sequence grown on demand

## Purpose

Creatures do not declare their packs; they inherit a (team, squad, group) triple from their
spawn record, and every creature sharing a triple is in one pack. This registry is the
mapping from triple to pack. It exists so that a creature can find its packmates without
any creature holding a list of them, and so that the pack outlives any individual member.

The storage is a nested sequence indexed directly by the three small integers — no hashing,
no search. That is the right shape: the identifiers are dense, small, and authored.

## State

```text
RECORD PackRegistry
  packs : list<list<list<optional<Pack>>>>    # indexed [team][squad][group]
```

**Invariants** — an index that has been reached is either a pack or empty; an index beyond
the current extent does not exist yet. Lookups assert all three indices are in range, so
looking up a pack for a triple that has never registered a member is a programming error,
not a runtime condition. The registry owns every pack and destroys them all at teardown.

## `register_member`

**Contract** — ensures a pack exists at the triple, growing the nesting as needed, and adds
the member to it. Growing fills intervening slots with nothing, so a spawn file that jumps
from group 0 to group 5 leaves four empty slots rather than four packs.

```text
FUNCTION register_member(team, squad, group, member)
  grow packs so that [team][squad][group] is addressable,
       filling every newly created intervening slot with empty
  IF packs[team][squad][group] is empty THEN create a pack there
  packs[team][squad][group].register(member)
```

**Notes** — the original writes the growth as four separate branches (no team, no squad, no
group, all present), which differ only in how much they resize. They collapse into one
grow-to-fit step and a rebuild should write it that way; the four-branch form has a real
consequence, though, which is that **the first two branches resize the outer levels
without preserving anything below them**. That is safe only because a triple's team index
is only ever seen for the first time when nothing below it exists — which holds for
authored spawn data and would not hold if triples were assigned at runtime.

## `find_pack`

**Contract** — two forms: by explicit triple, and by entity (which reads the triple off the
entity). Both assert the indices are in range and return whatever is stored, which may be
nothing.

## `update`

**Contract** — called with one entity per think. Runs the pack's coordination pass **only
when that entity is the pack's leader and the pack is active** (has a leader and at least
two living members). This is the sole driver of pack coordination.

```text
FUNCTION update(entity)
  pack = find_pack(entity)
  IF pack exists AND pack.is_active() AND pack.leader = entity THEN
    pack.coordinate()
```

**Invariants** — the coordination therefore runs at the leader's scheduler rate, and a pack
whose leader is far from the player coordinates rarely. That is the intended degradation and
is why no separate rate exists.

## `drop_references`

**Contract** — walks every pack in the registry and clears references to a destroyed object
from its goals and commands. Called once per destroyed object. Linear in the number of
packs, which is small.

## teardown

**Contract** — destroys every pack. Note that the registry itself is a process-wide global
created on first use and never destroyed, so this runs only at process exit; see the note in
[`ai_monster_squad_manager_inline.h`](ai_monster_squad_manager_inline.h.md).
