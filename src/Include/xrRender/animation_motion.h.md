# src/Include/xrRender/animation_motion.h

> The handle that names one animation: which motion bank, and which motion inside it.

**Needs** — _(none beyond the core types)_
**Used by** — [`KinematicsAnimated.h`](KinematicsAnimated.h.md) · [`animation_blend.h`](animation_blend.h.md) · [`aimers_base.h`](../../xrGame/aimers_base.h.md) · [`death_anims.h`](../../xrGame/death_anims.h.md)
**Tier floor** — T2: a value handle; it is packed into one 32-bit word because it is compared, sorted and copied in inner loops, but nothing outside the process ever sees the packing.

## Purpose

An animation is identified by a name in content and by a handle in code. Resolving the name is a map lookup over several banks; the AI, weapon and creature layers do it once at load and then pass the handle around for the rest of the session. This file is that handle.

It is its own file because it is included by both the shared animation data model and the game layer, and neither should have to pull in the other.

## State

```text
RECORD MotionID
  slot : int (16-bit)      # which motion bank
  index: int (16-bit)      # which motion within that bank
  # stored as one 32-bit word; invariant: the all-ones word means "no motion"
```

**Invariants**

- The all-ones 32-bit word is the invalid handle, and it is the default value: a handle is invalid until something sets it. There is no separate validity flag.
- Because validity is a reserved bit pattern, bank 65535 and index 65535 are jointly unrepresentable. In practice a model has a handful of banks and a few thousand motions per bank, so the reservation costs nothing.
- Handles order by their packed word — bank first, then index within the bank. Sorted containers of handles rely on this, and the ordering is therefore "grouped by bank", not "alphabetical by name".

## `MotionID`

**Contract** — a copyable value. Equality, inequality and ordering compare the packed word. `valid` asks whether it is not the reserved pattern; `invalidate` resets it. Constructing from a (bank, index) pair is the only way to make a valid one, and only the animation banks may do it.

```text
FUNCTION valid() -> bool
FUNCTION invalidate()
FUNCTION set(slot, index)
```

**Notes** — One behaviour here is deliberate and surprising: **moving a handle invalidates the source**. Move-construction and move-assignment copy the word and then reset the original to the invalid pattern, even though the type is a trivial value with nothing to release.

That is not an ownership convention — nothing is owned — it is a debugging aid turned into a rule: a handle that has been moved out of names an animation that the new owner is now responsible for, and leaving a live duplicate behind hides double-play bugs. A rebuild may keep the behaviour or drop it, but it must know it is there, because code that moves a handle and then reads the source will see an invalid handle where the original would have been fine.

The packing itself is incidental: a rebuild may store two fields and a flag. What is load-bearing is that the handle is **small, copied by value, compared as a unit, and has a reserved invalid state that is also its default** — several thousand of them are stored in the AI layer's tables and the default must cost nothing to produce.
