# src/xrMaterialSystem/GameMtlLib.cpp

> Reads the material half of the frozen library file, infers which game the file came from, supplies acoustic properties the shipped files never carried, and builds the dense pairwise table.

**Needs** — [`GameMtlLib.h`](GameMtlLib.h.md) · [`Common/FSMacros.hpp`](../Common/FSMacros.hpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — reached through its declarations in [`GameMtlLib.h`](GameMtlLib.h.md); callers name that, not this file.
**Tier floor** — T1: it reads a frozen chunked file, including one block copied in as a raw byte image, and computes a checksum over the file's bytes.

## Purpose

Three separable jobs share this file because all three run exactly once, at the moment
the library is read. The first is parsing a material record out of the chunked container.
The second is deciding, from the *shape* of what was parsed, which of the three shipped
games this data belongs to — a decision the file itself refuses to state. The third is
turning the sparse authored pair list into the square lookup table the running engine
indexes.

The split against [`GameMtlLib_Engine.cpp`](GameMtlLib_Engine.cpp.md) is not arbitrary:
that file holds everything that needs the audio device and the renderer, so that the
authoring tool can link this file without linking a game. A rebuild that has no separate
tool may merge them.

## State

```text
# Module-level: the one library instance. See GameMtlLib.h.

# File-local: a table of acoustic properties keyed by material name, about fifty
# entries covering every material the three shipped games use. It is a fallback
# for data that does not exist on disk — see `acoustics_for`.
```

## The library file format

**Contract** — `gamemtl.xr`, resolved against the game-data root, in the engine's
recursive chunked container: a chunk is a 32-bit identifier, a 32-bit payload length, and
the payload; a payload may itself be a sequence of chunks. Readers seek a chunk by
identifier, so the order chunks appear in is not part of the format. A chunk *sequence*
is read positionally instead, and position is what assigns a material its index.

```text
# Top level of gamemtl.xr
0x1000 VERSION   : int (16-bit)      # must be 1; anything else is refused outright
0x1001 AUTOINC   : int (32-bit) next_material_id
                   int (32-bit) next_pair_id
0x1002 MTLS      : sequence of material records, one per child chunk
0x1003 MTLS_PAIR : sequence of pair records, one per child chunk
```

```text
# One material record. MAIN, FLAGS, PHYSICS and FACTORS are required;
# the rest are present or absent per generation, and absence is meaningful.
0x1000 MAIN      : int (32-bit) id
                   text          name        # zero-terminated
0x1005 DESC      : text          description # optional
0x1001 FLAGS     : int (32-bit) flag bits
0x1002 PHYSICS   : real friction
                   real damping
                   real spring
                   real bounce_start_velocity
                   real bouncing
0x1003 FACTORS   : real shoot_factor
                   real bounce_damage_factor
                   real vis_transparency_factor
                   real snd_occlusion_factor
0x1004 FLOTATION : real flotation_factor      # optional; default 1
0x1006 INJURIOUS : real injurious_speed       # optional; default 0
0x1007 DENSITY   : real density_factor        # optional; presence => Clear Sky or later
0x1008 FACTORS_MP: real shoot_factor_mp       # optional; presence => Call of Pripyat
0x1009 ACOUSTICS : real absorption[3]         # optional; an extension, see below
                   real scattering
                   real transmission[3]
```

```text
# One pair record. Read in GameMtlLib_Engine.cpp.
0x1000 PAIR      : int (32-bit) material_0_id
                   int (32-bit) material_1_id
                   int (32-bit) pair_id
                   int (32-bit) parent_pair_id     # all-ones means none
                   int (32-bit) own_property_bits
0x1002 BREAKING  : text  # comma-separated sound names
0x1003 STEP      : text  # comma-separated sound names
0x1005 COLLIDE   : text  # comma-separated sound names
                   text  # comma-separated particle-effect names
                   text  # comma-separated decal shader names
```

**Notes** — three identifiers are burned and must never be reused: a pair-level flotation
chunk, an older collide chunk that carried a different payload, and an older material
"shootable" flag bit. The COLLIDE chunk packs three independent lists into one chunk
because it was extended in place rather than split, which is why its three strings are
read consecutively with no lengths or separators between them beyond the string
terminators. All numbers are little-endian and all reals are 32-bit; nothing in the file
is byte-swapped anywhere.

## `load_material(reader) -> Generation`

**Contract** — fills one material record from one chunk sequence and reports the newest
game generation the record proves. Fails hard on a missing required chunk: a truncated
library is a corrupt installation, not a recoverable condition, and guessing would produce
a level whose surfaces are silently wrong.

```text
FUNCTION load_material(reader) -> Generation
  generation <- shadow_of_chernobyl        # the floor: assume the oldest

  REQUIRE chunk MAIN     ; id <- int32 ; name <- string
  IF chunk DESC present  ; description <- string
  REQUIRE chunk FLAGS    ; flags <- int32
  REQUIRE chunk PHYSICS  ; read the five contact reals in order
  REQUIRE chunk FACTORS  ; read the four gameplay reals in order

  IF chunk FLOTATION present ; flotation_factor <- real
  IF chunk INJURIOUS present ; injurious_speed  <- real

  IF chunk DENSITY present
    density_factor <- real
    generation <- clear_sky                # the chunk is the evidence

  IF chunk FACTORS_MP present
    shoot_factor_mp <- real
    generation <- call_of_pripyat
  ELSE
    shoot_factor_mp <- shoot_factor        # one rule set, used for both

  acoustics <- acoustics_for(name, snd_occlusion_factor)
  RETURN generation
```

**Invariants** — a material that carries a multiplayer shoot factor also carries a
density; the generations are a ladder and later files never drop a chunk an earlier one
had. The loader does not check this, and a hand-edited file that violates it is reported
as Call of Pripyat.

**Notes** — the optional chunks are not optional *fields*; they are the format's only
record of which game shipped the file. Everything else about the three generations is
identical, which is why the version number stayed at 1 forever and why the engine has to
read the shape instead. Defaults for absent chunks are the record's own initial values
(no flotation resistance, no injury), chosen so an older library behaves as though the
feature it predates were switched off.

## `acoustics_for(name, occlusion)`

**Contract** — produces the seven acoustic reals for a material. If the record carried an
acoustics chunk, that is used verbatim. Otherwise a name-keyed table of authored values
supplies them, and if even the name is unknown, the entry for the generic default material
is used and the three absorption bands are then overwritten with the material's single
sound-occlusion factor.

```text
FUNCTION acoustics_for(name, occlusion) -> Acoustics
  IF the record carried an acoustics chunk
    RETURN it as stored

  IF name is in the authored table
    RETURN that entry

  a <- the table's default entry
  a.absorption <- [occlusion, occlusion, occlusion]   # flat across all bands
  RETURN a
```

**Notes** — this is the one place where the module invents data rather than reading it.
No shipped library carries an acoustics chunk; the chunk and the table are both additions
by this engine, so that a three-band reverb model has per-surface numbers to work with
without requiring anyone to re-author `gamemtl.xr`. The fifty-odd table entries are tuned
values with no derivation available: concrete absorbs little and transmits almost nothing,
foliage absorbs and scatters heavily, a water surface transmits nearly everything, and the
"death" material is a perfect transmitter that absorbs nothing at all. Nothing in this
repository consumes the acoustics fields yet, so the numbers are unverified against any
behaviour — treat them as a seed, not as a specification. The degenerate fallback — one
occlusion number smeared across three frequency bands — is honest about being a
placeholder.

A rebuild may store this table as data rather than compiling it in; the reason it is not
data here is that it must be available before any configuration file is known to exist.

## `load_library`

**Contract** — reads the whole library from the game-data root, leaving the module ready
to answer queries. Tolerates a missing file by logging and leaving the library empty,
which lets a tool run without game data; every later query against an empty library fails.
Refuses a file whose stated version is not the single supported value, again leaving the
library empty. Requires that the library not already be loaded. Allocates; does not block
beyond the file read; is not safe to run concurrently with anything that reads the library.

```text
FUNCTION load_library()
  IF gamemtl.xr does not exist under the game-data root
    LOG the missing path ; RETURN

  REQUIRE both collections are empty
  open the file

  REQUIRE chunk VERSION ; stated <- int16
  IF stated is not the supported version
    LOG and RETURN                       # no partial load, no guessing

  checksum <- hash over the file's bytes

  REQUIRE chunk AUTOINC ; next_material_id, next_pair_id <- two int32
  reserve both collections to those counts

  generation <- shadow_of_chernobyl
  FOR EACH child chunk OF chunk MTLS
    append a new material ; generation <- newer of (generation, load_material(child))
  library_generation <- generation

  FOR EACH child chunk OF chunk MTLS_PAIR
    append a new pair ; load_pair(child)

  build_pair_table()
```

**Invariants** — a material's index is its ordinal in the MTLS sequence, so the sequence
order is load-bearing even though the container is otherwise order-free. The authoring
counters are read for their side effect of sizing the collections; they are upper bounds,
not exact counts, because deleting a material in the tool does not decrement them.

**Notes** — the library is loaded serially during game-module startup and explicitly not
in parallel with other load work. Two failures forced that: concurrent loading corrupts on
one platform outright, and on another it intermittently produces water surfaces that
render as mercury — a symptom of the pair table being read while it is still being filled.
The lesson for a rebuild is that the table must be fully built before any consumer can
observe the library, and the cheapest way to guarantee that is to publish the library only
once loading has returned.

## `build_pair_table`

**Contract** — expands the sparse authored pair list into the dense square table. Runs
once, immediately after the pairs are read, while nothing else can observe the library.

```text
FUNCTION build_pair_table()
  n <- material_count
  pair_table <- n * n empty slots
  FOR EACH pair IN pairs
    a <- index_of_material_with_id(pair.material_0)
    b <- index_of_material_with_id(pair.material_1)
    pair_table[a * n + b] <- pair
    pair_table[b * n + a] <- pair          # both orders, one record
```

**Invariants** — after this runs, every slot is either empty or holds a record whose two
materials are the slot's two indices. Slots stay empty for the great majority of
combinations and every consumer must handle that.

**Notes** — this is where pair records stop being addressed by material *id* and start
being addressed by material *index*: the file speaks ids, the running engine speaks
indices, and this loop is the only translation between them. The translation is a linear
scan per pair, so the build is quadratic in the worst case; it happens once, on a list of
a few hundred pairs, and is not worth optimizing.

## Library checksum

**Contract** — a 32-bit hash standing for the identity of the loaded file, exposed for the
level loader's collision-model cache. Two libraries that differ anywhere must produce
different values; the value need not be stable across engine versions, only within one
run's write-then-read cycle.

**Notes** — what matters to a rebuild is the *role*, not the algorithm: level loading
rewrites every collision triangle's material number from a library id to a library index,
so a cached collision model is only valid against the library it was built from. The
checksum is the invalidation key. The original computes it over a byte range that begins
after the version chunk's payload and runs for the file's full length, which reaches past
the end of the buffer by the size of that prefix — a latent over-read that happens to
produce a stable value for a given file. A rebuild should hash the whole file and nothing
more.

## Debug pair naming

**Contract** — in diagnostic builds a pair can render itself as the two material names it
joins, for assertion messages. The assertions that use it — "this pair has no collide
sounds" — are the only way a data error in the library is ever attributed to a specific
combination, so a rebuild that drops it loses the only diagnostic the format has.
