# src/xrGame/relation_registry.cpp

> Stores and clamps the two goodwill maps, and owns the process-wide registry they live in.

**Needs** — [`relation_registry.h`](relation_registry.h.md) · [`relation_registry_defs.h`](relation_registry_defs.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`character_community.h`](character_community.h.md) · [`character_reputation.h`](character_reputation.h.md) · [`character_rank.h`](character_rank.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`game_type.h`](game_type.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a keyed registry over the alife object set

## Purpose

The storage half of the relation system. The derived attitude formula is in
[`relation_registry_inline.h`](relation_registry_inline.h.md); this file provides what that
formula reads: the personal goodwill map, the faction goodwill map, their clamps, and the
lifetime of the registry holding them.

## State

```text
GLOBAL relation_registry  : optional<AlifeRegistry of RelationData>   # created on demand
GLOBAL fight_registry     : optional<list<FightData>>                 # created on demand
GLOBAL spot_names         : optional<RelationMapSpots>                # created on demand
```

**Invariants**

- All three are process-wide and lazily created. A new game or a loaded save destroys and
  rebuilds them through the single teardown call, which is the only place they are released.
- **Creating the relation registry asserts that the game is single player.** The entire
  social model is a single-player feature; multiplayer never touches it. A rebuild that
  reaches this code from a multiplayer path has a bug elsewhere.
- Creating the relation registry also triggers the one-time load of the attack point tables
  ([`relation_registry_actions.cpp`](relation_registry_actions.cpp.md)). Binding that to
  registry creation rather than to level load is what guarantees the tables exist before
  anyone can score an action.
- A `RelationData` record is created on first *write* for a character; reads of an absent
  record answer neutral without creating one. This keeps the registry proportional to the
  number of characters with opinions rather than to the world's population.

## `relation_registry` / `clear_relation_registry`

**Contract** — the accessor creates the registry on first use and returns it thereafter; the
teardown releases all three globals. Nothing outside the teardown may destroy them, because
every other holder keeps a bare reference.

## `goodwill` — get, set, change

**Contract** — the stored personal opinion of one character about another, by identifier.
Reading an absent entry answers neutral. Writing **clamps to the configured personal limits**
before storing and creates the owner's record if needed. The change form is read, add,
write-with-clamp, so repeated small changes saturate at the limit rather than accumulating
invisibly.

**Invariants** — the clamp is the reason a player cannot grind a faction's opinion
arbitrarily high or low through repeated small actions. It applies on every write, including
the change form, so the limit is on the *stored* value and not on any single delta.

## `force_set_goodwill`

**Contract** — sets the personal goodwill such that the *total* attitude comes out at the
requested value, by subtracting the two faction contributions before storing. Requires both
parties to have server records that carry a faction; logs a script-level error and does
nothing if either does not.

```text
FUNCTION force_set_goodwill(from, to, wanted)
  from_record = alife.object(from); to_record = alife.object(to)
  IF either is not a trading character THEN
    log_script_error("cannot convert object")
    RETURN
  stored = wanted
         - community_goodwill(from_record.faction, to)
         - faction_relation(from_record.faction, to_record.faction)
  personal[from][to] = stored
```

**Invariants** — it compensates for the two faction terms and **not** for the rank and
reputation terms, so the resulting attitude still differs from the requested value by those
two. The name promises more than it delivers; a rebuild should either compensate for all
four derived terms or name the call for what it compensates.

**Notes** — the result is not clamped, unlike the ordinary setter. A large faction penalty
can therefore push the stored personal value outside the range the ordinary setter enforces,
where a subsequent ordinary write will silently snap it back. That inconsistency is real and
is a good argument for the rebuild picking one write path.

## `community goodwill` — get, set, change

**Contract** — a faction's opinion of one character, stored on the *character*. Same shape as
the personal trio and the same absent-reads-neutral rule, clamped against its own configured
limits, which are a separate pair from the personal ones.

**Notes** — there is deliberately no "a character's opinion of a faction". A character's
stance toward a faction is derived: his opinion of its members plus that faction's standing
with his own. A rebuild adding the missing direction would duplicate information the attitude
formula already composes.

## `community relation` — get and set

**Contract** — pure delegation to the faction table, which holds a square matrix of
faction-to-faction standings loaded from configuration and mutated by quests. Routed through
the registry so that callers have one place to ask about relations regardless of whether
the parties are characters or factions.

## `rank relation` / `reputation relation`

**Contract** — private helpers that turn a raw rank or reputation value into its band and
look the pair up in that attribute's matrix. Both are pure table lookups; the interesting
decision — banding rather than interpolating — is described in
[`relation_registry_inline.h`](relation_registry_inline.h.md).

## `clear_relations`

**Contract** — empties one character's personal and faction maps, if it has a record. Does
not create a record for a character that has none, and does not remove that character from
anyone *else's* maps.

**Notes** — the asymmetry is a real leak: after a character is cleared, every other character
still holds an opinion of it keyed by identifier. Since identifiers are a 16-bit space that
is recycled across a long game, a newly spawned character can inherit the reputation of a
dead one. The shipped games do not hit it because entity counts stay well below the wrap
point, but a rebuild with a long-running world must either sweep the maps or key opinions by
something that is not recycled.

## `spot_name`

**Contract** — answers the map-marker name for a relation type, building the four-entry table
on first use. Out-of-range types answer the neutral marker.

## `RelationData` — clear, load, save

**Contract** — see [`relation_registry_defs.h`](relation_registry_defs.h.md). The
implementation is the generic map serializer applied to each map in declaration order.

## `Relation` construction

**Contract** — a fresh relation is neutral. There is no uninitialized state, which is what
makes the absent-reads-neutral rule consistent with the stored one.
