# src/xrGame/memory_space_script.cpp

> The creature-memory vocabulary as scripts see it: the eight memory record types and the danger record, with their two enumerations.

**Needs** — [`memory_space.h`](memory_space.h.md) · [`danger_object.h`](danger_object.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`GameObject.h`](GameObject.h.md) · [`entity_alive.h`](entity_alive.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a binding table

## Purpose

Every name registered here is frozen by conformance criterion 10 — shipped scripts read
these fields by name. The file is a declaration of that surface and contains no logic beyond
four adapters.

## State

`Stateless.`

## `script_register`

**Contract** — registers the memory and danger vocabulary into the script virtual machine.
Runs once at startup, from the game's central binding registration.

The exported surface, with the script-visible name first:

- `rotation` — heading and pitch, writable. Exported unconditionally even though nothing in
  a memory record currently produces one (see [`memory_space.h`](memory_space.h.md)).
- `object_params` — a memory's geometric snapshot: `level_vertex`, `position`. Read-only.
- `memory_object` — the base record: `level_time`, `last_level_time`. Read-only. The other
  timestamps in the record's declaration are compiled out and therefore not exported, which
  means **the exported field set is decided by a build switch** — a rebuild must export
  exactly these two or shipped scripts break.
- `entity_memory_object` / `game_memory_object` — the same record over a living entity and
  over any object: `object_info` (what I remember about it), `self_info` (where I was), and
  `object()`. Two names for one shape because the two flavours are distinct types to the
  binding layer.
- `hit_memory_object` — extends the entity flavour with `direction`, `bone_index`, `amount`.
- `visible_memory_object` — extends the object flavour with nothing. The per-squad-member
  visibility query exists on the record and is **not exported**; the binding is written and
  commented out. Scripts can hold a visual memory and cannot ask who can currently see it.
- `memory_info` — the merged record: `visual_info`, `sound_info`, `hit_info`. This is what
  a script iterating a creature's memory actually receives.
- `sound_memory_object` — `type()` and `power`.
- `not_yet_visible_object` — `value` and `object()`: a sighting still accumulating
  recognition, which is how a script can react to being noticed before being seen.
- `danger_object` — a recorded threat: `position()`, `time()`, `type()`, `perceive_type()`,
  `object()`, `dependent_object()`, and equality. With two enumerations nested inside it.

**Notes** — four small adapters exist between the record and the binding, and each one is
solving the same problem: **the script layer sees a game object only through its facade**,
and a record holds the client object. Each adapter asks the client object for its facade.
One of them additionally reports nothing when the danger's dependent object is not a game
object at all. A rebuild whose script layer can address the object directly deletes all four.

The `position` adapter on the danger record fails loudly rather than quietly on a missing
record, where the others only assert in development builds. Scripts reach for a danger's
position in code paths that were crashing obscurely; this converts that to a diagnosable
failure.

## The danger enumerations

**Contract** — both are part of the frozen script surface and both are authored-behaviour
vocabulary rather than implementation detail.

```text
ENUM danger_type          # what happened
  bullet_ricochet         # a round struck near me
  attack_sound            # I heard weapon fire
  entity_attacked         # someone I can see is under attack
  entity_death            # someone died in front of me
  entity_corpse           # I found a body that is still fresh
  attacked                # I was hit
  grenade                 # a live grenade is near me
  enemy_sound             # I heard something I consider hostile

ENUM danger_perceive_type # which sense told me
  visual
  sound
  hit
```

**Notes** — the two enumerations are orthogonal and both are recorded, because the brain's
response depends on both *what* happened and *how it learned*. Hearing a shot and being shot
are the same event class with very different urgency, and a corpse found by sight decays
differently from one inferred from a sound. The danger weighting that consumes this pair is
in the creature brains.

`entity_corpse` is distinguished from `entity_death` by whether the creature witnessed it.
Both are in the enumeration because a body discovered later is still a threat signal, just a
weaker and older one.
