# src/xrGame/DamagableItem.cpp

> Turns a continuous health value into a small number of discrete damage levels, and fires one step per level crossed so that visible damage stages are never skipped.

**Needs** — [`DamagableItem.h`](DamagableItem.h.md)
**Used by** — [`DamagableItem.h`](DamagableItem.h.md)
**Tier floor** — T3: arithmetic over one scalar

## Purpose

A mixin for things that degrade in stages rather than continuously — a car whose windows
crack then shatter then whose engine dies, a destructible prop that loses pieces. The
problem it solves is that the *effect* of damage is discrete (swap a model, detach a
part, start a smoke effect) while the *cause* is a continuous health value, and a single
large hit must not skip the intermediate effects: a car struck once for full health must
still lose its windows on the way to being wrecked.

It solves this by remembering the highest level already applied and, on every hit,
replaying every level between that and the new one. The implementor supplies only
"what health am I at" and "apply stage N"; the mixin owns the staging.

## State

```text
RECORD Damagable
  levels_num    : int     # how many damage stages exist; from configuration
  max_health    : real    # health at which the item is undamaged
  level_applied : int     # highest stage whose effect has already run
```

Invariants: `level_applied` never decreases through the hit path — degradation is
one-way, and an item that regains health does not un-break. `level_applied` is clamped to
`levels_num`, and a fully applied item ignores further hits entirely. Both counters are
sentinel-valued until `Init` runs, so `Init` must precede any hit; nothing checks this at
run time.

## `Init`

**Contract** — establishes the health scale and the number of stages, and resets the
applied stage to zero (undamaged). Called when the owning entity loads its configuration.

## `DamageLevel`

**Contract** — maps current health to a stage index: zero at full health, `levels_num` at
zero health, linear between. Negative health reads as zero. The result is clamped, so a
health value above maximum still reports stage zero rather than a negative stage.

```text
FUNCTION damage_level() -> int
  h = max(health(), 0)
  level = floor((1 - h / max_health) * levels_num)
  IF level < levels_num THEN RETURN level
  RETURN levels_num
```

**Notes** — the clamp on the high side exists because health can reach exactly zero, which
would otherwise produce an index one past the last stage. The low side is *not* clamped
against health exceeding maximum; no caller does that.

## `DamageLevelToHealth`

**Contract** — the inverse: the health value at which a given stage begins. Used to set an
item's health from a saved or authored stage rather than the other way round, so that a
prop can be spawned already half-wrecked.

## `HitEffect`

**Contract** — runs the staging. Applies every stage strictly above the last applied one
up to and including the stage the current health implies, in ascending order, each
through the implementor's apply hook. Applying a stage records it as the new high-water
mark, so the sequence is idempotent across repeated calls at the same health.

```text
FUNCTION hit_effect()
  target = damage_level()
  FOR EACH level IN (level_applied + 1) .. target
    apply_damage(level)          # implementor's hook; also advances level_applied
```

**Invariants** — no stage is ever applied twice and no stage between the old and new
levels is skipped, whatever the size of the hit.

## `RestoreEffect`

**Contract** — replays every stage from the first up to the one the current health
implies, *ignoring* the high-water mark. This is the load path: an item restored from a
save has its health already set and no effects applied, so every stage it has passed
must be re-run to rebuild the visible state. Distinct from `HitEffect` precisely because
its starting point is "nothing applied" rather than "whatever was applied".

## `ApplyDamage`

**Contract** — the implementor's hook. The base does one thing: record the stage as
applied. An implementor overrides it, does its own visible work, and is expected to call
through — the bookkeeping is the base's, and an implementor that forgets will replay
stages forever.

## `CDamagableHealthItem`

**Contract** — the variant for implementors that have no health of their own: it owns the
scalar. `Hit` subtracts damage, floors it at zero and runs the staging; once the last
stage is applied it becomes inert and further hits cost nothing. `SetHealth` writes the
value directly, for the load path, and is expected to be followed by `RestoreEffect`.

**Notes** — the two classes exist because some damagable things already have a health
value owned by their condition record and some do not. In a rebuild this is one type
parameterized by a health accessor.
