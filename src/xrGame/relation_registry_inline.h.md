# src/xrGame/relation_registry_inline.h

> The attitude formula: five contributions summed into one signed number, then cut into friend, neutral and enemy by two configured thresholds.

**Needs** — [`relation_registry.h`](relation_registry.h.md) · [`character_community.h`](character_community.h.md) · [`character_rank.h`](character_rank.h.md) · [`character_reputation.h`](character_reputation.h.md)
**Used by** — [`relation_registry.h`](relation_registry.h.md)
**Tier floor** — T2: four map lookups and an addition per query, run per creature per perception update

## Purpose

This is where the social simulation actually happens. Everything else in the relation system
stores numbers; this file decides what they mean. The queries are generic over any party
that can report an identifier, a faction, a rank and a reputation — which lets the same
formula answer for a live stalker, for an offline server record, and for the actor.

## State

`Stateless.` Three thresholds are read from configuration once and cached for the process:
the goodwill values that *represent* enemy, neutral and friend, and the two attitude
thresholds that *classify* into them. Those are different numbers with different jobs and
confusing them is the easiest mistake here.

## `attitude`

**Contract** — computes the signed opinion one party holds of another. Sums five
contributions and returns the total; larger is friendlier. Reads four stored or derived
values and does not write. Requires both parties to be able to report faction, rank and
reputation.

```text
FUNCTION attitude(from, to) -> int
  personal   = stored_goodwill(from.id, to.id)              # the only part anyone sets directly
  reputation = reputation_table(from.reputation, to.reputation)
  rank       = rank_table(from.rank, to.rank)

  community_to_person = NEUTRAL
  IF from.faction EXISTS THEN
    community_to_person = stored_community_goodwill(from.faction, to.id)

  community_to_community = NEUTRAL
  IF from.faction EXISTS AND to.faction EXISTS THEN
    community_to_community = faction_table(from.faction, to.faction)

  RETURN personal + reputation + rank + community_to_person + community_to_community
```

**Invariants**

- The five terms are **added, not weighted**. Their relative influence is set entirely by
  the magnitudes the configuration assigns each table, which is why the shipped data's
  personal-goodwill clamp range and the faction table's range matter to each other. A
  rebuild that introduces weights will need to re-tune every shipped table.
- The faction's own opinion of the target is consulted through `from`'s faction, but it is
  *stored on the target* — see [`relation_registry_defs.h`](relation_registry_defs.h.md) for
  why the community map faces backwards.
- Rank and reputation contribute through lookup tables indexed by *band*, not by raw value:
  each of rank and reputation is first bucketed into a named tier and the table is a
  tier-by-tier matrix. Small changes in a raw value therefore do nothing until they cross a
  band boundary, which is what keeps the social layer from flickering.
- A party with no faction contributes neutral for both faction terms rather than skipping
  them, so a factionless character is not systematically friendlier or more hostile.

## `relation type` (from one party to another)

**Contract** — cuts the attitude number into the three-valued view the rest of the game uses.
Below the neutral threshold is enemy; below the friend threshold is neutral; at or above it
is friend. The special "no goodwill" sentinel short-circuits to neutral.

```text
FUNCTION relation_type(from, to) -> {ENEMY, NEUTRAL, FRIEND}
  a = attitude(from, to)
  IF a IS the no-goodwill sentinel THEN RETURN NEUTRAL
  IF a < neutral_threshold THEN RETURN ENEMY
  IF a < friend_threshold  THEN RETURN NEUTRAL
  RETURN FRIEND
```

**Notes** — the sentinel check is defensive: the stored goodwill accessor never returns it,
because a missing entry already reads as neutral. It is the residue of an earlier design in
which "never met" was distinguishable, and a rebuild that wants that distinction back has to
restore it in the storage layer, not here.

## `relation between` (two parties, symmetric)

**Contract** — the *mutual* relation, used wherever a fight or an alliance needs both sides
to agree. Asks the one-directional question in both directions and takes the **worse** of the
two answers.

```text
FUNCTION relation_between(a, b) -> {ENEMY, NEUTRAL, FRIEND}
  IF either direction says ENEMY   THEN RETURN ENEMY
  IF either direction says NEUTRAL THEN RETURN NEUTRAL
  RETURN FRIEND
```

**Invariants** — hostility is contagious and friendship is not. One party deciding the other
is an enemy makes the pair enemies; friendship requires both. This asymmetry is why shooting
a neutral makes him an enemy immediately rather than after he decides to reciprocate, and it
is load-bearing for how a faction turns on the player.

## `set relation type`

**Contract** — the inverse of classification: given a desired three-valued relation, write
the stored personal goodwill to the configured representative value for that relation. Used
by scripts and by the dialogue system when a quest declares a relation outright.

**Invariants** — setting a relation and then reading it back does **not** reliably return
what was set, because the read sums in faction, rank and reputation contributions the write
did not account for. The force-set variant in
[`relation_registry.cpp`](relation_registry.cpp.md) exists precisely to compensate for the
faction terms; this one does not. A rebuild should pick one policy and name the two
operations for what they do — *set my personal opinion* versus *make the total come out
here*.

**Notes** — the neutral value's configuration key is misspelled in the shipped data
(`goodwill_neutal`) and the code reads it under the misspelling, as it does again for the
neutral threshold. The typo is in the shipped configuration files and is therefore frozen:
a rebuild must read the misspelled key or find nothing.
