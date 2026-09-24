# src/xrGame/Level_load.cpp

> The game layer's two hooks into level loading — what must exist before the geometry loads, what must be built after it — plus the material-index remap that makes the shipped collision database agree with the shipped material library.

**Needs** — [`Level.h`](Level.h.md) · [`ai_space.h`](ai_space.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`level_sounds.h`](level_sounds.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it rewrites material indices in place inside the collision database's triangle array, which is a memory image the query code walks.

## Purpose

The engine loads a level's geometry, collision and visibility data; the *game* has its own
things to load around that, and they cannot all happen at the same moment. Navigation data
must exist before anything that references a navigation vertex; static particles, sounds
and level scripts must not exist until the game mode is known, because they are filtered
by it. This file is the two halves of that, plus the one transformation the game applies
to engine-owned data: remapping the collision database's per-triangle material tag.

## State

Stateless per call; the static particles it creates are owned by the level.

## `load_before_geometry`

**Contract** — Runs before the level's geometry and collision load. Brings up the
navigation data for this level when the world is *not* under alife control — an alife
session loads its navigation through the simulator instead, because the simulator owns the
cross-level graph and must not have a second loader racing it. On a rendering client with
no alife but with a game graph, it also loads the level's authored patrol paths, which are
named point sequences the script layer hands to creatures. Always succeeds.

```text
FUNCTION load_before_geometry() -> bool
  show "loading ai objects"
  IF single player AND no alife simulator AND the level has navigation data AND a host is known
    load navigation for this session
  IF rendering AND no alife simulator AND a game graph exists AND the level has patrol data
    read patrol paths from the level, keeping the raw block for later parsing
  RETURN success
```

**Notes** — Patrol paths are kept as an unparsed block rather than decoded here: they are
only needed when a script asks for one by name, and most levels' paths are never touched.

## `load_after_geometry`

**Contract** — Runs after the geometry is in place and after the game mode exists.
Instantiates everything in the level that the game layer owns and the renderer or the
scripts will reach for. Order within it is mostly free; the one hard requirement is that
the static-particle list is empty on entry, because this routine appends and a second call
would double the world's particle emitters.

**Invariants** — On entry the static-particle list is empty. On exit the environment's
clock has been set from the game's own time, so the first drawn frame already has the
right sky.

```text
FUNCTION load_after_geometry() -> bool
  ASSERT static particle list is empty

  # 1. static particle emitters, filtered by game mode
  IF the level has a static-particle file
    FOR EACH record IN the file's chunk sequence
      IF the first chunk is a bare version number, remember it and CONTINUE
      IF version > 0 THEN read the game-mode mask ELSE mask means "all modes"
      read emitter name and its placement transform
      raise the placement 1 cm                 # authored flush with the floor; lifts it off
      IF this game mode is in the mask
        create the emitter, seat it at the transform with zero velocity, start it looping
        append to the static particle list

  IF rendering
    # 2. authored ambient sound placements
    load the level's sound manager
    # 3. acoustic geometry: what occludes and what reverberates
    IF the level has a sound-environment file, hand it to the audio device
    IF the level has a sound-occlusion file, hand it to the audio device
    # 4. the random one-off ambience played around the player
    IF configuration has a random-sounds section
      create one source per key in it
      arm the next random sound far in the future, disabled until gameplay enables it
    # 5. fog volumes: read and discarded (see Notes)

  IF rendering
    # 6. this level's own script process, replacing any previous level's
    drop the level script process
    scripts = level configuration's level_scripts/script, or empty
    create a fresh level script process running those scripts

  apply the anti-cheat clamp (below)
  set the environment clock from the game's time and time factor
  RETURN success
```

**Notes** — The fog-volume file is parsed and every value thrown away. The format is read
correctly — a version tag, a count, a transform per volume and a count of sub-volumes each
with its own transform — but nothing consumes it, because the renderer that used volumetric
fog was cut. A rebuild should skip the file entirely; the parse is recorded here only so a
reader who finds the file knows what is in it.

The random-sound timer is armed 50 seconds ahead and the system starts disabled, so the
first random ambience cannot fire during the load or the first moments of play.

The script process is *replaced*, not added to: a level's scripts are scoped to that level
and must not leak into the next one.

## `remap_collision_materials`

**Contract** — Rewrites the material tag on every triangle of the loaded collision
database, turning the authored material *identifier* into an index into the currently
loaded material library, and caching two of that material's flags directly on the triangle.
Fails fatally on a triangle whose material is not in the library — a level authored against
a different material set cannot be silently mis-rendered.

**Invariants** — Only *static* materials may appear on level geometry; dynamic materials
belong to objects. The translation table therefore contains only static materials plus a
fallback, and the fallback is bound to the out-of-range identifier so that unassigned
triangles land on the library's `default` material rather than failing.

```text
FUNCTION remap_collision_materials(triangles)
  table : list<(material_id, library_index)>
  table += (out-of-range id, index of "default")
  FOR EACH material IN library, with its index
    IF material is not dynamic
      table += (material.id, index)

  IF table is small                  # under 128 entries
    lookup = linear scan             # scanning beats sorting at this size
  ELSE
    sort table by id; lookup = binary search

  FOR EACH triangle
    entry = lookup(triangle.material)
    IF none: FAIL WITH "game material not found"
    triangle.material         = entry.library_index
    triangle.suppress_shadows = library[entry.library_index].suppress_shadows
    triangle.suppress_wallmarks = library[entry.library_index].suppress_wallmarks
```

**Notes** — The two flags are copied onto the triangle rather than looked up at query time
because they are read once per ray hit, in the renderer's hot path: a decal placement and a
shadow test each ask "does this surface take marks" for every hit, and one extra indirection
there is measurable. This is a cache, and a rebuild that keeps the library lookup instead is
correct but slower.

The 128-entry threshold is the crossover the authors chose between scanning and sorting; it
is a tuning number, not a correctness boundary, and nothing depends on its exact value.

## `remap_collision_materials_by_name`

**Contract** — The same remap, but driven by a supplied identifier-to-*name* map rather
than by the identifiers the collision file carries. Used when the collision database was
built earlier and the material library has since been edited: names are stable across
edits, numeric identifiers are not. Fails fatally on an unknown name.

## `serialize_material_signature` / `verify_material_signature`

**Contract** — Writes, and later checks, a checksum of the whole material library
alongside a cached collision database. A cached database whose signature disagrees with the
current library is rejected and rebuilt, because the indices baked into it by the remap
above are only meaningful against the library that produced them.

## `apply_anticheat_clamp`

**Contract** — In a non-debug build of a non-single-player session, forces the physics
time factor back to one. The time factor is a console variable a player could otherwise use
to run the local simulation slower or faster than the server's; single player is exempt
because there is nobody to cheat.
