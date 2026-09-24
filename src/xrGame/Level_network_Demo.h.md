# src/xrGame/Level_network_Demo.h

> The demo recorder's slice of the level's own declaration: fields and methods spliced directly into the level class, implemented in [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md).

**Needs** — [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md)
**Used by** — [`Level.h`](Level.h.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md)
**Tier floor** — T1: it declares two records whose byte layout is the demo file format, written with no padding

## Purpose

This is not a header in the usual sense: it has no include guard, no class of its own and
no self-contained meaning. It is a fragment textually spliced into the middle of the level
class's declaration, so that the demo system's state and methods become members of the
level without the level's own header carrying them. A rebuild that can express partial
class definitions, extensions or mixins should use whichever of those its language has; a
rebuild that cannot should simply put these members on the level and delete the file.

What survives the split is the *grouping*: the demo recorder owns a coherent slice of the
level's state and nothing outside it should reach into those fields.

Substance is in [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md), which also documents
the file format the two records below describe.

Exported units:

- `DemoHeader` — the four clock values at the head of a demo file. **Packed with no
  padding**, because it is written and read as a raw memory image.
- `DemoPacket` — the per-message record: time offset, receive stamp, length, then the
  message bytes. Also packed.
- `PrepareToSaveDemo` / `StartSaveDemo` / `SaveDemoHeader` / `SaveDemoInfo` / `SavePacket` /
  `StopSaveDemo` — the recording side.
- `PrepareToPlayDemo` / `LoadDemoHeader` / `StartPlayDemo` / `RestartPlayDemo` /
  `StopPlayDemo` / `LoadPacket` / `SimulateServerUpdate` — the playback side.
- `IsDemoPlay` / `IsDemoSave` / `IsDemoPlayStarted` / `IsDemoPlayFinished` /
  `IsDemoSaveStarted` — the state predicates. Each of the first two is defined as *one flag
  set and the other clear*, so a state that is somehow both answers no to both.
- `GetDemoPlayPos` / `GetDemoPlaySpeed` / `SetDemoPlaySpeed` — playback transport.
- `SpawnDemoSpectator` / `SetDemoSpectator` / `GetDemoSpectator` — the fake entity a viewer
  inhabits.
- `GetMessageFilter` / `GetDemoPlayControl` — lazily created helpers.
- `CatchStartingSpawns` / `MSpawnsCatchCallback` — the one-shot observer that remembers
  where the first spawn message sits in the file, so playback can be restarted.
- `GetDemoInfo` — the match summary read from or written to the file's reserved block.
