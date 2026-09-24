# src/xrMaterialSystem/GameMtlLib.h

> The whole vocabulary of surface materials: the per-material record, the pairwise-interaction record, the query surface every other subsystem reaches through, and the frozen chunk identifiers of the library file.

**Needs** — [`xrCore/xrstring.h`](../xrCore/xrstring.h.md) · [`xrCore/_flags.h`](../xrCore/_flags.h.md) · [`xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md) · [`xrSound/Sound.h`](../xrSound/Sound.h.md) · [`Include/xrRender/WallMarkArray.h`](../Include/xrRender/WallMarkArray.h.md) · [`Include/xrRender/RenderFactory.h`](../Include/xrRender/RenderFactory.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`WallMarkArray.h`](../Include/xrRender/WallMarkArray.h.md) · [`DetailManager_Decompress.cpp`](../Layers/xrRender/DetailManager_Decompress.cpp.md) · [`ModelPool.cpp`](../Layers/xrRender/ModelPool.cpp.md) · [`dxWallMarkArray.cpp`](../Layers/xrRender/dxWallMarkArray.cpp.md) · [`xr_efflensflare.cpp`](../xrEngine/xr_efflensflare.cpp.md) · [`Actor_Feel.cpp`](../xrGame/Actor_Feel.cpp.md) · [`Car.cpp`](../xrGame/Car.cpp.md) · [`ClimableObject.cpp`](../xrGame/ClimableObject.cpp.md) · [`CustomRocket.cpp`](../xrGame/CustomRocket.cpp.md) · [`Explosive.cpp`](../xrGame/Explosive.cpp.md) · [`GamePersistent.cpp`](../xrGame/GamePersistent.cpp.md) · [`HUDTarget.cpp`](../xrGame/HUDTarget.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](../xrGame/Level_bullet_manager_firetrace.cpp.md) · [`PHMovementControl.cpp`](../xrGame/PHMovementControl.cpp.md) · _and 22 more_
**Tier floor** — T1: it fixes the chunk identifiers and the byte image of the acoustics block of a shipped, frozen file, and it exposes a dense lookup table indexed by a 16-bit surface number that a collision triangle stores inline.

## Purpose

This file is the substance of the module, not a declaration surface, and the twins are
split accordingly: the data records, the flag vocabulary, the file's chunk identifiers,
the version ladder and the entire query surface live here, because in the original they
are defined here and implemented inline here. The reading of those records from the
library file lives in [`GameMtlLib.cpp`](GameMtlLib.cpp.md) and
[`GameMtlLib_Engine.cpp`](GameMtlLib_Engine.cpp.md).

There are two records and one singleton. A **material** describes one surface — concrete,
glass, a monster's body, a bullet. A **material pair** describes what happens when one
surface meets another, and holds only the things that need two surfaces to be decided:
which sound plays, which particle effect spawns, which decal is stamped. The **library**
owns both collections and the dense table that turns a pair of surfaces into a pair record
in constant time.

## State

```text
RECORD Material
  id            : int                 # authored auto-number, stable in the file
  name          : text                # "materials\concrete", "objects\small_box"
  description   : text                # authoring note; no runtime reader
  flags         : int (32-bit, a bit set)

  # physics: contact parameters, combined pairwise at contact time
  friction              : real        # default 1
  damping               : real        # default 1
  spring                : real        # default 1
  bounce_start_velocity : real        # default 0
  bouncing              : real        # default 0.1

  # gameplay factors
  flotation_factor      : real        # 0..1, 1 = no resistance to passing through
  shoot_factor          : real        # 0..1, armour value a bullet must exceed
  shoot_factor_mp       : real        # same, multiplayer rule set
  bounce_damage_factor  : real        # 0..100, collision damage scale
  injurious_speed       : real        # health lost per second while standing on it
  vis_transparency      : real        # 0..1, 1 = AI and lens flares see straight through
  snd_occlusion_factor  : real        # 0..1
  density_factor        : real        # penetration resistance per metre travelled

  acoustics : Acoustics

RECORD Acoustics
  absorption   : list<real>  # exactly 3, one per frequency band, low to high
  scattering   : real
  transmission : list<real>  # exactly 3, matching bands

# Invariants the loader and the callers rely on:
#   name is unique across the library; lookups by name are ASCII case-insensitive.
#   id is unique; index (position in the loaded array) is NOT id and the two are
#     routinely confused by anyone reading the source for the first time.
#   index fits in 14 bits, because a collision triangle packs it inline alongside two
#   flags and a 16-bit sector id in one 32-bit word. Caps a level at 16384 materials.
#   flotation_factor < 1 is mirrored in the slow-down flag; injurious_speed > 0 is
#     mirrored in the injurious flag. The loader does not recompute either — the
#     authoring tool wrote both, and a rebuild that writes the file must keep them
#     consistent or the physics and the damage tick disagree with the numbers.
```

```text
RECORD MaterialPair
  material_0 : int          # material ids, not indices
  material_1 : int
  id         : int          # authored auto-number
  parent_id  : int          # the pair this one was cloned from; none == -1
  own_props  : int (bit set) # which media below are this pair's own vs inherited

  breaking_sounds   : list<Sound>     # at most 6
  step_sounds       : list<Sound>     # at most 6
  collide_sounds    : list<Sound>     # at most 6
  collide_particles : list<text>      # at most 4, particle effect names
  collide_marks     : Wallmarks       # at most 4 decal shaders

RECORD Sound        # opaque: one handle into the audio device seam
RECORD Wallmarks    # opaque: one renderer-created collection of decal shaders

# Invariant: the pair is unordered. A pair authored as (A,B) answers a query for
# (B,A). Nothing in the pair record distinguishes which surface is the striker.
```

```text
RECORD MaterialLibrary
  materials       : list<Material>       # order defines index
  pairs           : list<MaterialPair>   # sparse, as authored
  pair_table      : list<optional<MaterialPair>>  # dense, size = |materials|^2
  next_material_id : int                 # authoring counters, read and kept
  next_pair_id     : int
  checksum        : int (32-bit)         # identity of the loaded file
  version          : Generation

# Invariant: pair_table[a * |materials| + b] and pair_table[b * |materials| + a]
#   name the same pair record, or both are empty. Empty is normal and common:
#   most surface combinations are not authored, and every caller must have a
#   fallback.
```

## Surface flags

**Contract** — a 32-bit bit set attached to each material, authored in the editor and
read straight from the file. The bit *positions* are frozen by the shipped library, not
the names; two positions are dead (an early "shootable" bit and an early "walk-on" bit)
and must stay dead so the remaining bits keep their places.

```text
ENUM SurfaceFlag            # bit position
  breakable          = 0    # authoring-only: no engine reader
  bounceable         = 2    # both surfaces must set it before a contact bounces
  skidmark           = 3    # authoring-only: no engine reader
  bloodmark          = 4    # a wound on this surface stamps a blood decal
  climable           = 5    # a character may climb it; foot IK ignores it
  passable           = 7    # collision is skipped entirely: bushes, grass, foliage
  dynamic            = 8    # belongs to a model, not to level geometry
  liquid             = 9    # contact drag is applied as buoyancy, not as friction
  suppress_shadows   = 10   # baked into the collision triangle at level load
  suppress_wallmarks = 11   # baked into the collision triangle at level load
  actor_obstacle     = 12   # passable to everything except the player's body
  no_ricochet        = 13   # bullets never deflect off it
  injurious          = 28   # derived from injurious_speed > 0
  shootable          = 29   # authoring-only: no engine reader
  transparent        = 30   # authoring-only: no engine reader
  slow_down          = 31   # derived from flotation_factor < 1
```

**Notes** — `passable` and `actor_obstacle` together express "the player cannot walk
through this bush but a bullet, a monster and a grass blade can". `dynamic` partitions
the library: dynamic materials belong to model bones, static ones to level geometry, and
only the static half participates in the level-load remapping described in
[`GameMtlLib.cpp`](GameMtlLib.cpp.md).

## Pair property flags

**Contract** — a bit set on each pair saying which of its five media lists the pair owns
rather than inherits from its parent pair. It exists for the authoring tool, which lets a
designer clone a pair and override one list. The running engine never consults it: the
tool resolves inheritance before writing, so every pair in the shipped file is complete.
Bit positions are again frozen with two dead slots (an early flotation bit and an early
collide-sounds bit).

```text
ENUM PairProperty          # bit position
  breaking_sounds   = 1
  step_sounds       = 2
  collide_sounds    = 4
  collide_particles = 5
  collide_marks     = 6
```

**Notes** — a rebuild that only *reads* the shipped library can drop this field and the
parent link with it, keeping the four bytes each occupies in the file. A rebuild that also
writes the library must keep both, because a designer's clone-and-override workflow is the
only reason the field exists.

## `Generation`

**Contract** — which of the three shipped games this library came from. This is the
module's most surprising export: the material library is the engine's *game detector*.
The library file carries a version number that the original developers never incremented
in eight years, so the generation is inferred from which optional chunks the material
records actually contain, and the answer then selects rule variants far outside this
module — how armour-piercing is read from a cartridge, which sign convention an artefact's
damage immunities use, what a bone's default hit fraction is, how a stalker's armour
subtracts from incoming damage.

```text
ENUM Generation             # ordered; comparisons are meaningful
  shadow_of_chernobyl = -2  # virtual: inferred, never written
  clear_sky           = -1  # virtual: inferred, never written
  unknown             =  0
  call_of_pripyat     =  1  # the only value that appears on disk
```

**Notes** — the negative virtual values exist so that the three generations sit in one
ordered scale with the single real on-disk number, letting every caller ask
"at least Clear Sky?" with one comparison. A rebuild is free to use three ordered names
and no negative numbers; what must survive is the ordering and the inference rule.

## Library file identity

**Contract** — the library is a single file named `gamemtl.xr`, resolved against the
game-data root. There is exactly one library and it is a process-wide singleton, loaded
once when the game module starts and released when it shuts down. Every consumer reaches
it as a global.

**Notes** — the singleton is a cycle-breaker, not a design preference: physics, sound, the
renderer, the AI and the game all need the table and none of them owns it. A rebuild
should inject it, and the twins note at each use site what is actually being reached for.

## Material lookup

**Contract** — the library answers four questions about a single material, and the
distinction between the two identifiers is the thing to get right.

- **by name → index**, and **by name → id**: an ASCII case-insensitive scan of the name
  list. Used once per entity at spawn, once per level at load; never in a frame.
- **by id → index**: a scan of the id list. Used only while remapping level geometry.
- **by index → material**: a direct array index. This is the hot path — every collision
  contact, every bullet hit, every footstep, every grass blade takes it.

```text
FUNCTION material_index_of(name) -> int (16-bit)
  # FAILS if absent. Callers that tolerate absence ask for the id instead,
  # which yields the sentinel "no material" value.

FUNCTION material_at(index) -> Material
  # Total: index is required to be in range. Out of range is a programming error,
  # not a data error, because indices only ever come from this library.
```

**Notes** — the sentinel for "no material" is the all-ones value in both widths: a 32-bit
one for ids and a 16-bit one for indices. Entities carry the 16-bit sentinel while they
have no footing, and the damage tick tests for it before asking a surface how fast it
hurts.

## `material_pair_for(index_a, index_b)`

**Contract** — the load-bearing operation of the whole module. Given two material
*indices* in either order, return the pair record for that combination, or nothing. Total
in the sense that any two valid indices answer; the answer is frequently empty, because
the library authors only the combinations that need a sound or a decal.

```text
FUNCTION material_pair_for(a, b) -> optional<MaterialPair>
  # One indexed read into a dense square table. No search, no hashing.
  RETURN pair_table[b * material_count + a]
```

**Invariants** — the table is square and symmetric, so argument order never changes the
answer. It is built once at load and never mutated, which is what makes it safe to read
from the physics thread, the sound thread and the render thread without a lock.

**Notes** — the table costs one pointer per ordered combination, which for a shipped
library of a few dozen materials is a few kilobytes: the memory is irrelevant and the
constant-time answer is not, because it is taken inside the physics contact callback. A
rebuild that stores the sparse pair list and searches it will pay for that choice in the
inner loop of collision. The symmetry is a real design decision and a limitation: the
engine cannot say "a bullet hitting concrete sounds different from concrete hitting a
bullet", and where it needs that asymmetry — bullets, footsteps — it encodes the striker
as its own material rather than as a direction.

## Library enumeration and identity

**Contract** — the library exposes its material collection for iteration and its count,
used by the level loader to build a remapping table; and two scalars decided at load:

- **generation**, as above;
- **checksum**, a 32-bit hash standing for "this exact library file". The level loader
  writes it into the cached collision model and refuses a cache whose checksum differs,
  because triangle material numbers are rewritten against the library's ordering and a
  reordered library silently corrupts every surface in the level.

## `unload`

**Contract** — releases both collections and the table, and clears the checksum so a
stale cache can never match an unloaded library. Every sound handle a pair holds is
released as part of the pair's own teardown; see
[`GameMtlLib_Engine.cpp`](GameMtlLib_Engine.cpp.md).

**Notes** — the ordering matters and is the reason this is an explicit operation rather
than something that happens when the last reference drops: pair records hold audio-device
handles, and the audio device is torn down by the same shutdown sequence. The library must
release its handles while the device is still alive.

## `save`

**Contract** — declared as part of the surface on all three records, and implemented
nowhere in this repository. The writing side of the format belongs to the authoring tool.

**Notes** — a rebuild that only ships a game can omit it. A rebuild that wants to author
libraries must reconstruct it from the reading side, which is fully specified in the two
implementation twins; the chunk ordering a writer chooses is free, because readers seek
chunks by identifier.
