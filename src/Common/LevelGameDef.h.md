# src/Common/LevelGameDef.h

> The chunk identifiers and record layouts for the authored marker objects a level carries — spawn points, environment modifiers and patrol paths.

**Needs** — _(none)_
**Used by** — [`patrol_path.cpp`](../xrAICore/Navigation/PatrolPath/patrol_path.cpp.md) · [`patrol_path_storage.cpp`](../xrAICore/Navigation/PatrolPath/patrol_path_storage.cpp.md) · [`Environment_misc.cpp`](../xrEngine/Environment_misc.cpp.md) · [`game_sv_artefacthunt.cpp`](../xrGame/game_sv_artefacthunt.cpp.md) · [`game_sv_capture_the_artefact.cpp`](../xrGame/game_sv_capture_the_artefact.cpp.md)
**Tier floor** — T1: it names identifiers in a frozen chunked binary format that the level editor already wrote.

## Purpose

Besides geometry and entities, a level file carries small authored annotations: places the
player or an item may appear, volumes that override the weather, and named paths that
creatures walk. These have no runtime class of their own at this layer — they are records
in the level's chunked container, and this file is where their identifiers and encodings
are fixed.

## State

```text
ENUM PointType : int (32-bit)      # the kinds of authored point marker
  respawn_point     = 0
  environment_mod   = 1
  spawn_point       = 2

ENUM WayType : int (32-bit)        # the kinds of authored path
  patrol_path       = 0

ENUM RespawnPointRole              # what a respawn point is for; stored in one byte
  actor_spawn       = 0
  artefact_spawn    = 1
  item_spawn        = 2
  # the value 255 is reserved as the end-of-list terminator in editor tables

ENUM EnvironmentModField           # bit flags: which weather fields this volume overrides
  view_distance     = bit 0
  fog_color         = bit 1
  fog_density       = bit 2
  ambient_color     = bit 3
  sky_color         = bit 4
  hemisphere_color  = bit 5
```

**Invariants**

- Chunk identifiers are computed, not listed: a point chunk's identifier is `0x2000` plus
  the point type, a way chunk's is `0x1000` plus the way type. The bases are chosen far
  apart so the two families never collide as the enumerations grow, and the arithmetic means
  **the enumeration values are part of the file format** — reordering them silently
  renumbers chunks in files already on disk.
- A respawn point's role is stored in one byte, so the enumeration may never exceed 255
  members.
- The two marker kinds also carry a text name used by the editor's object browser;
  those spellings are fixed as `$rpoint` and `$env_mod`.

## Record layouts

**Contract** — the annotation chunks are nested chunked containers: an outer chunk per family,
then one numbered sub-chunk per object.

```text
RECORD RespawnPoint                 # one sub-chunk of the respawn-point chunk
  position   : vector of 3 real
  rotation   : vector of 3 real
  team_id    : int (8-bit)
  role       : int (8-bit)          # a RespawnPointRole
  reserved   : int (16-bit)         # present so the record is 4-byte aligned

RECORD WayObject                    # one sub-chunk of a way chunk, itself chunked
  version : int (16-bit)            # current format version is 0x0013
  name    : text                    # zero-terminated
  kind    : WayType
  points  : list<WayPoint>          # prefixed by a 16-bit count
  links   : list<WayLink>           # prefixed by a 16-bit count

RECORD WayPoint
  position : vector of 3 real
  flags    : int (32-bit)
  name     : text

RECORD WayLink
  from        : int (16-bit)        # index into the point list
  to          : int (16-bit)
  probability : real                # weight for choosing among outgoing links
```

**Invariants** — the sub-chunk identifiers inside a way object are a small fixed set: version,
points, links, kind, name. A way object's point count is 16-bit, so a patrol path may not
exceed 65535 points — far beyond anything authored.

A link's probability makes a patrol path a weighted directed graph rather than a cycle: a
creature at a point chooses its next point among the outgoing links by weight. That is the
whole behavioural content of this format.

## Notes

The enumerations carry a trailing `max` member used as a bound when iterating the families.
It is not a chunk identifier and must not be written.

A table mapping the respawn-point roles to human-readable names is present but commented
out, with a note that it needs a home with an implementation file. Nothing reads it; the
editor's own tables carry the strings.

The file is named for the *game* definitions of a level while its contents are the *editor*
annotations. The split between this and [`LevelStructure.hpp`](LevelStructure.hpp.md) is
arbitrary — both are chunk vocabularies for the same container — and a rebuild is free to
merge them.
