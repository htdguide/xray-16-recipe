# src/xrGame/danger_explosive.cpp

> The record of one live grenade a creature has noticed, and the rule that lets it be matched by object identifier.

**Needs** — [`danger_explosive.h`](danger_explosive.h.md) · [`GameObject.h`](GameObject.h.md) · [`Explosive.h`](Explosive.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: an identity comparison

## Purpose

A grenade in flight is tracked separately from the general danger set because the reaction
is different in kind: a creature does not take cover from a grenade, it runs, and it must
not react twice to the same one. This record is what makes "the same one" decidable.

It is kept as a small standalone record rather than as a danger location because it needs
three references the danger set does not carry — the explosive itself, the same thing as a
game object, and which creature is already reacting.

## State

```text
RECORD DangerExplosive
  grenade     : optional<Explosive>    # the explosive behaviour
  game_object : optional<GameObject>   # the same entity as a world object
  reactor     : optional<Stalker>      # who has taken responsibility for reacting
  time        : int (ms)               # when it was noticed
```

Invariant: **if `grenade` is present then `game_object` is present**. The two are two views
of one entity, held separately because the reaction needs both the explosive's parameters
and the object's position, and neither view can be derived from the other without a cast.
An empty record (no grenade) is legal and means "no grenade is being tracked".

**Notes** — a rebuild in a language where one entity can be asked for either interface
directly should keep a single reference and drop the invariant entirely. The pair exists
only because the original could not name both at once.

## Comparison against an entity identifier

**Contract** — matches this record against an entity identifier. Yields false when no
grenade is tracked; otherwise compares the tracked entity's identifier. This is how a
creature asks "am I already reacting to the grenade I was just told about", given only the
identifier that arrived in the notification.

```text
FUNCTION matches(object_id) -> bool
  IF grenade is none: RETURN false
  RETURN grenade.as_game_object().id == object_id
```

**Notes** — the identifier, not the reference, is the match key because notifications travel
as identifiers: a grenade may be reported by a squadmate that saw it, and the recipient has
only the number. The reference comparison is available too (see
[`danger_explosive_inline.h`](danger_explosive_inline.h.md)) and is used when the caller
already holds the object.

## `CDangerExplosive` construction and reference comparison

**Contract** — both are defined in
[`danger_explosive_inline.h`](danger_explosive_inline.h.md).
