# src/Common/LevelStructure.hpp

> The chunk vocabulary of a compiled level and the seven generations of the AI navigation node record, which is the most tightly bit-packed structure in the engine.

**Needs** — [`GUID.hpp`](GUID.hpp.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — [`Light_DB.cpp`](../Layers/xrRender/Light_DB.cpp.md) · [`r__sector.cpp`](../Layers/xrRender/r__sector.cpp.md) · [`game_level_cross_table.h`](../xrAICore/Navigation/game_level_cross_table.h.md) · [`level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`level_graph_manager.h`](../xrAICore/Navigation/level_graph_manager.h.md) · [`level_graph_space.h`](../xrAICore/Navigation/level_graph_space.h.md) · [`xr_area.cpp`](../xrCDB/xr_area.cpp.md) · [`IGame_Level.cpp`](../xrEngine/IGame_Level.cpp.md) · [`Level_SLS_Save.cpp`](../xrGame/Level_SLS_Save.cpp.md) · [`alife_spawn_registry_header.cpp`](../xrGame/alife_spawn_registry_header.cpp.md) · [`alife_spawn_registry_header.h`](../xrGame/alife_spawn_registry_header.h.md) · [`SoundRender_Scene.cpp`](../xrSound/SoundRender_Scene.cpp.md)
**Tier floor** — T1: navigation nodes are read as a memory image out of a mapped file, and their fields are unaligned bit runs addressed by byte offset.

## Purpose

Two things live here because both are read straight off disk at level load. First, the
identifiers of the top-level chunks in a compiled level and the fixed headers that precede
its geometry, collision and navigation data. Second — and this is the bulk of the file —
the node record of the AI navigation mesh, in every version the engine can still read.

The navigation mesh is a dense grid of walkable cells covering a level, on the order of a
million of them. At that count the per-node record's size is a first-order decision: each
byte costs a megabyte of resident memory and a megabyte of load time. That is why the
record is bit-packed rather than laid out in fields, and why it has been re-packed six
times across the games and community forks this engine supports.

## State

```text
ENUM LevelChunk : int (32-bit)       # top-level chunks of the compiled level file
  header            = 1
  shaders           = 2              # material pass descriptions, by name
  visuals           = 3
  portals           = 4              # portal polygons
  dynamic_light     = 6              # note: 5 is absent and unused
  glows             = 7
  sectors           = 8
  vertex_buffer     = 9              # static geometry
  index_buffer      = 10
  progressive_mesh  = 11             # collapse information, chiefly for trees

ENUM SectorChunk : int (32-bit)      # inside one sector's record
  portals           = 1
  geometry_root     = 2

ENUM SaveChunk : int (32-bit)        # inside a saved game
  description       = 1              # the level's name
  server_state      = 2

ENUM BuildQuality : int (16-bit)
  draft = 0, high = 1, custom = 2
```

```text
RECORD LevelHeader                   # 8-byte aligned
  compiler_version : int (16-bit)
  build_quality    : int (16-bit)    # a BuildQuality

RECORD CollisionHeader               # precedes the static collision triangle soup
  version    : int (32-bit)
  vertices   : int (32-bit)
  faces      : int (32-bit)
  bounds     : box of 6 real

RECORD NavigationHeader              # precedes the navigation node array
  version    : int (32-bit)
  count      : int (32-bit)          # number of nodes
  cell_size  : real                  # horizontal extent of one node, in world units
  height     : real                  # vertical extent of one node
  bounds     : box of 6 real
  guid       : Guid                  # matched against the level's other outputs
```

**Invariants**

- Every level output of one compiler run carries the same identity — see
  [`GUID.hpp`](GUID.hpp.md). An older navigation header exists without the identity field
  and is still readable; the identity was added, not replaced.
- The whole family is **frozen**: these headers and the node array are read by pointing a
  record at a mapped file region, little-endian, no byte swapping. See
  [§5 Data and persistence](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).

## Version gate

```text
ENUM NavigationVersion : int (8-bit)
  shadow_of_chernobyl = 8    # release, 2005
  priquel             = 9    # early Clear Sky builds on the older engine
  clear_sky_and_cop   = 10   # release Clear Sky and Call of Pripyat
  wide_links          = 11   # community: node links widened to 25 bits
  wide_position       = 12   # community: node position widened to 6 bytes
  current             = 13   # 26-bit links, rudimentary light data removed

  lowest_accepted     = shadow_of_chernobyl
  produced            = current
```

**Contract** — a navigation file, a cross-reference table or a game graph header whose version
falls outside `lowest_accepted .. produced` is rejected with a message naming the file and
both versions. Rejection rather than best-effort is the rule throughout: the engine refuses
a mismatch rather than guessing.

**Invariants** — the version is stored as a 32-bit value in two of the three places and as a
single byte in the third, so the enumeration may never exceed 255. That constraint is
asserted at build time.

## The navigation node record

**Contract** — one node is a cell of the walkable surface. It stores four links to neighbouring
nodes, a quantized position, a packed surface plane, and cover values describing how
exposed the cell is from each of four directions. Everything is packed to the bit.

```text
RECORD NavigationNode              # current generation; 25 bytes, unaligned
  link_bits : 13 bytes             # four neighbour indices, 26 bits each, packed
                                   #   end to end with a 2-bit stagger (see below)
  cover_high: CoverQuad            # 2 bytes: four 4-bit values
  cover_low : CoverQuad            # 2 bytes: four 4-bit values
  plane     : int (16-bit)         # the cell's surface orientation, quantized
  position  : NodePosition         # 6 bytes

RECORD CoverQuad                   # 2 bytes
  values : four int (4-bit)        # exposure from each of four directions, 0..15

RECORD NodePosition                # 6 bytes, current generation
  planar : int (32-bit)            # a single index into the level's cell grid;
                                   #   x = planar / row_length, z = planar MOD row_length
  height : int (16-bit)            # quantized vertical offset within the level's bounds
```

**Invariants**

- **A node is addressed by index, not by pointer.** The link field holds neighbour indices
  into the node array, which is why the array can be mapped and used in place.
- **The link width bounds the level.** 26 bits gives 67 million nodes. The earlier widths —
  23 bits (8.4 million) and 25 bits (33 million) — were each raised because a community
  level exceeded them. A rebuild choosing its own width should note that the width is the
  level size limit.
- **The four links share their bytes.** Each link is read as a 32-bit window starting at a
  byte offset and shifted: link 0 at byte 0 shifted 0, link 1 at byte 3 shifted 2, link 2
  at byte 6 shifted 4, link 3 at byte 9 shifted 6. That pattern — advance three bytes,
  advance two bits — is exactly what packs four 26-bit fields into 13 bytes with no waste.
  Writing a link must preserve the bits of its neighbours, so a write is read-modify-write
  over the same window.
- **Reading a link reads past the record's end.** The window for link 3 starts at byte 9
  and is 32 bits wide, reaching byte 12 — the last of the 13 — which is fine, but the same
  technique on the earlier 12-byte generations reads one byte beyond. It is safe only
  because nodes are contiguous in a larger mapped array. A rebuild that reads nodes
  individually must bound the read itself.
- **Position packing is a grid index, not coordinates.** The horizontal position is a
  single number that divides and remainders by the level's row length into cell
  coordinates; the row length lives in the level graph, not in the node. Vertical position
  is a 16-bit quantization across the level's height.
- Cover values are 4 bits each, four per word, giving 16 exposure levels in each of four
  directions. The current generation stores two such words — a high and a low sample — where
  the oldest generation stored one; the conversion duplicates the single value into both.

## Generational differences

**Contract** — six node layouts and three position layouts are readable. They differ only in
widths, and each newer one can be assigned from the older:

```text
generation 7 : 21 bytes — 23-bit links in 12 bytes, ONE cover word, 5-byte position
generation 10: 23 bytes — 23-bit links in 12 bytes, two cover words, 5-byte position
generation 11: 24 bytes — 25-bit links in 13 bytes, two cover words, 5-byte position
generation 12: 25 bytes — 25-bit links in 13 bytes, two cover words, 6-byte position
generation 13: 25 bytes — 26-bit links in 13 bytes, two cover words, 6-byte position  [current]

position, 5-byte : horizontal packed into 3 bytes (max 16.7 million cells),
                   vertical into 2 — the two read as overlapping windows of a
                   byte array, masked
position, 6-byte : horizontal a full 32-bit value, vertical 16 bits
position, 3-byte : a signed x, unsigned y, signed z triple — the editor's uncompressed
                   form, never written to a level file
```

**Invariants** — the 25-bit generations pack their four links with a *1-bit* stagger (byte
offsets 0, 3, 6, 9), and the 23-bit generations with an irregular one (byte offsets 0, 2,
5, 8 with shifts 0, 7, 6, 5). The irregularity in the oldest is what packing 4×23 bits into
12 bytes forces. Each is a different arithmetic and must be reproduced exactly to read
existing levels.

Only the most recent layout is ever *written*. The rest exist to be read and converted
forward at load time, which is how one engine loads three games and two community forks.

## Notes

The editor builds against the uncompressed 3-byte position and the engine against the
6-byte one, chosen at build time. The two are not interchangeable at runtime, which means
editor-built and engine-built binaries genuinely differ in this structure — an unusual and
fragile arrangement that a rebuild should replace with one in-memory form plus explicit
encode and decode.

Three further version constants sit at the end of the file and belong to the level
compiler rather than the engine: the newest level format the engine will read (18), the
format the tools emit (14), the collision format (4) and a version for a collision cache
added later (1). The gap between 18 and 14 is deliberate — the engine reads newer files
than the shipped tools produce — but nothing in the source explains what the four
intervening versions changed.

A comment credits the two-cover-word change to a specific upstream commit, and the
25-bit and 26-bit widenings to a named community fork. Those attributions are the only
record of *why* each generation exists, and they are worth preserving in any rebuild's
documentation because the format numbers are otherwise arbitrary.
