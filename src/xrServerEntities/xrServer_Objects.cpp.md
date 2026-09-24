# src/xrServerEntities/xrServer_Objects.cpp

> Four small records that most other records are built out of: a volume, a saved ragdoll pose, an empty observer, and a bare navigation position.

**Needs** — [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`ShapeData.h`](ShapeData.h.md) · [`PHNetState.h`](PHNetState.h.md) · [`game_base_space.h`](game_base_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: four on-disk field groups.

## Purpose

These are mixins and degenerate cases, not entities a designer places. Their value is that
every restrictor, zone, vehicle, corpse and lamp in the game is partly made of them, so their
byte layouts appear inside a dozen other records.

## `CSE_Shape`

**Contract** — the volume mixin: a list of shapes, each a sphere (centre plus radius) or an
oriented box (a full transform), tagged by a type byte.

```text
RECORD Shape
  type : int (8-bit)     # 0 = sphere, 1 = box
  data : sphere(centre : real x 3, radius : real) OR box(transform : real x 12)
```

```text
FUNCTION cform_read(packet)
  count = read 8-bit                      # at most 255 shapes in one volume
  REPEAT count TIMES
    type = read 8-bit
    IF type == 0 THEN read sphere ELSE read box
```

**Invariants**

- **The shape count is one byte.** A volume authored with more than 255 shapes cannot be
  represented; no shipped level has one.
- **The sphere is read as a raw memory image in the binary path and field-by-field in the
  text path.** A packet can be backed by a text configuration file instead of bytes — the
  level editor writes records that way — and a raw copy has no meaning there. The
  *decision* is that the record format has two encodings of the same fields, binary and
  textual, and a few places must branch. A rebuild that parses field-by-field everywhere
  makes the branch disappear, which is legitimate as long as the binary field order is
  preserved exactly.
- **The box is stored as a full 12-real transform**, not centre-plus-half-extents, so it
  carries rotation and non-uniform scale together.

## `CSE_PHSkeleton`

**Contract** — the mixin for a record that can be a ragdoll: it remembers whether the
physics is active, whether a stored pose exists, whether the record should be saved at all,
and the per-bone pose itself.

```text
RECORD PhysicsSkeletonMixin
  startup_animation : text          # borrowed from the record's own visual mixin
  flags             : int (8-bit)   # active | spawn-copy | saved-data | do-not-save
  source_id         : int (16-bit)  # the record this one broke off, 0xFFFF = none
  saved_bones       : bytes         # present iff the saved-data flag is set
```

**Invariants**

- **The saved-data flag is the length discriminator.** The bone pose is present in the stream
  exactly when the flag says so; there is no count.
- **The animation name is written by this mixin but stored on the visual mixin**, which means
  a record cannot have a physics skeleton without also having a visual. The mixin asserts
  this on every read and write.
- **`need_save` is the do-not-save flag inverted**, and it is what every physics-bearing
  class returns from its own save predicate. A door that was never touched is not worth a
  save record; one that was broken is. This is the mechanism by which a save file stays a
  few hundred kilobytes in a level with thousands of props.
- **Writing the pose twice in a row yields different results**, and the code says so and does
  not know why. It is left as-is deliberately: clearing the saved-data flag after a write
  was tried and broke something. **Unrecovered** — a rebuilder should assume the pose write
  must be idempotent and treat any divergence as a bug in their own version.

**`load`** — the save path reads the flag byte and then the pose unconditionally, and resets
the source identifier to "none" rather than restoring it: a broken-off piece does not
remember its parent across a save.

## `CSE_Spectator`

**Contract** — the multiplayer free-camera observer. Its payload is empty in all four
directions. It is the one class exempted from the "a record must have a payload" assertion.

## `CSE_Temporary`

**Contract** — a record whose entire state is one navigation vertex identifier (32-bit),
written and read with no version gate. Update is empty.

## `CSE_AbstractVisual`

**Contract** — a base record plus a model. Its payload is the visual mixin's fields followed
by the startup animation name, in that order. No update payload.

**Notes** — the startup animation is written *after* the visual fields and by this class
rather than by the visual mixin, because the mixin's own layout was frozen at version 104
and the animation was already being written by several classes at different offsets by then.
Copying that ordering is mandatory; explaining it is not possible beyond "history".
