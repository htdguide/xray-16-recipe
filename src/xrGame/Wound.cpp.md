# src/xrGame/Wound.cpp

> One accumulated injury on one bone of a creature: how much of each damage type it has taken, how it heals, and how it survives a save or a network update.

**Needs** — [`Wound.h`](Wound.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`hit_immunity.h`](hit_immunity.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`Wound.h`](Wound.h.md); callers name that, not this file.
**Tier floor** — T2: a small numeric record with a quantized wire form

## Purpose

A creature does not carry a single health number for damage-display purposes; it carries
a list of wounds, each attached to a bone. The wound is what drives the blood-drip
particles, the bleeding rate and the visible injury decals, so it has to remember *where*
it is and *what kind* of damage made it — a burn and a bullet wound on the same bone are
different wounds in effect, and the record keeps them apart by carrying one magnitude per
damage type rather than one total.

It is a separate file because it is a value type shared by every damageable entity and by
the save and network paths, with no dependency on the entity that owns it.

## State

```text
RECORD Wound
  bone            : int (16-bit)   # the skeleton bone this wound sits on
  particle_bone   : int (16-bit)   # bone the drip particle plays from; "none" sentinel when absent
  particle_name   : text           # empty when this wound shows no particle
  magnitudes      : list<real>     # exactly one entry per damage type, index == damage type
  drop_time       : real           # seconds until the next blood drop; owner's clock, not ours
  to_be_destroyed : bool           # owner's mark that this wound should be dropped next sweep
```

Invariants:

- `magnitudes` has exactly as many entries as there are damage types, always, from
  construction; a missing damage type is a zero entry, never an absent one. Index *is* the
  damage-type tag, which is what lets the wire form be a fixed-length run.
- Every magnitude lies in `[0, WOUND_MAX]` where `WOUND_MAX` is 10. The ceiling is the
  quantization range of the wire form, so it is not a tuning value: exceeding it would
  make a saved wound read back wrong. Every write clamps.
- `particle_bone` may differ from `bone`: the injury is recorded where the hit landed but
  the blood is drawn from wherever looks right on the model.
- The destruction mark and the drop timer are owned by the wound but *driven* by the
  creature that holds it; nothing in this file ever reads them.

## `construct(bone)`

**Contract** — a wound on the given bone with every damage magnitude zero, no particle,
and no pending drop. Allocates the per-damage-type run once, up front, because it is
indexed by tag and never grown.

## `save` / `load`

**Contract** — writes and reads the wound's persistent part: the bone index and the run of
magnitudes. The particle name, the particle bone, the drop timer and the destruction mark
are *not* persisted — they are presentation state the owner reconstructs from the
magnitudes on restore.

**Invariants** — the bone index is written as 8 bits even though it is held as 16, which
caps a wound-bearing skeleton at 256 bones; the magnitudes are written as 8-bit
quantizations of the range `[0, WOUND_MAX]`, so a saved wound is coarse to about one part
in 255 of the maximum. Both are frozen by the save and network formats.

```text
FUNCTION save(writer)
  writer.write_byte(bone)
  FOR EACH type IN damage_types
    writer.write_quantized_real(magnitudes[type], low: 0, high: WOUND_MAX, bits: 8)

FUNCTION load(reader)
  bone = reader.read_byte()
  FOR EACH type IN damage_types
    magnitudes[type] = reader.read_quantized_real(low: 0, high: WOUND_MAX, bits: 8)
    REQUIRE 0 <= magnitudes[type] <= WOUND_MAX
```

**Notes** — the damage-type count is part of the format. Adding a damage type changes the
length of every saved wound, so the enumeration is effectively frozen alongside the save
version.

## `AddHit`

**Contract** — folds a hit of a given magnitude and damage type into this wound, clamped
into the legal range. Damage of different types never merges: two types on one bone stay
two independent magnitudes on one wound record.

## `TotalSize`

**Contract** — the sum over every damage type. This is the number used to decide whether a
wound still exists and how prominent its visual is.

## `TypeSize`

**Contract** — the magnitude of one damage type alone.

## `BloodSize`

**Contract** — the sum of exactly the two damage types that bleed: the plain wound type
and the fire-wound type. This is the quantity that drives blood-drip particles and the
bleeding health drain, which is why it is a named concept rather than a caller-side sum —
burns and chemical damage do not bleed, and the choice of which types do is a gameplay
decision, not an arithmetic one.

## `Incarnation`

**Contract** — heals the wound. Given a healing amount and a floor, subtracts the amount
from *every* damage type independently and snaps any type that falls below the floor to
zero. A wound that is already entirely zero is left zeroed and returns immediately.

**Invariants** — after the call no magnitude is negative, because anything below the floor
becomes zero and the floor is non-negative.

```text
FUNCTION heal(amount, min_wound_size)
  IF total_size() is zero THEN
    clear every magnitude to 0      # cheap idempotent path for the common already-healed case
    RETURN

  FOR EACH type IN damage_types
    magnitudes[type] = magnitudes[type] - amount
    IF magnitudes[type] < min_wound_size THEN
      magnitudes[type] = 0
```

**Notes**

- Healing is *absolute*, not proportional: every damage type loses the same amount per
  tick regardless of how large it is, so a small wound closes first and a large one takes
  proportionally longer. The name and the surrounding comment describe a proportional
  scheme, and the proportional term survives in the source commented out — so the
  intended design and the shipped behaviour differ. A rebuild must copy the *shipped*
  behaviour, because regeneration rates in the configuration data were tuned against it.
- The floor exists so that a wound converges to exactly zero in finite time instead of
  decaying asymptotically and leaving an invisible sliver that keeps the wound record —
  and its particle — alive forever.
