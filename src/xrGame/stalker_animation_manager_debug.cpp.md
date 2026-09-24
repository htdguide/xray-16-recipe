# src/xrGame/stalker_animation_manager_debug.cpp

> Checked-build instrumentation that counts, per animation and per transition between animations, how often each is played — so a modeller can find the motions the engine never selects and the blends it takes constantly.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two counting tables and two sorted reports

## Purpose

The animation tables are indexed by literal positions in authored name lists, across six
files, with no way to tell from reading the code which entries are ever reached. This is the
answer: a process-wide tally of what actually plays, and of what blends into what, dumped on
request.

Entirely compiled out of shipped builds. It is nonetheless load-bearing knowledge for a
rebuild, because it is the only evidence available that a given table entry matters.

## State

```text
RECORD AnimationStats                       # process-wide, keyed by (motion, motion set)
  frame_count : int                         # frames this animation was the one playing
  start_count : int                         # times it started

RECORD BlendStats                           # process-wide, keyed by (from, to) animation pair
  count : int                               # times this transition was taken
```

**Invariants** — animations are keyed by the pair (motion name, motion set name), because a
motion name alone is not unique: the same name appears in several sets, and which set a
stalker's model resolves it from is exactly the thing being investigated. Keys are compared
by interned-string identity rather than by content, which makes the comparison a pointer
compare at the cost of depending on the string interner's guarantee.

## `add_animation_stats`

**Contract** — record one frame of whichever channel is currently authoritative, following
the same priority ladder the update uses: script if it has an animation, else global if it
has one, else the torso and legs together. Called once per stalker per frame in checked
builds.

**Invariants** — the ladder must match the update's, or the tally attributes frames to a
channel that was suppressed. The head channel is deliberately excluded — it is commented out
rather than absent — because it plays essentially continuously and would swamp the report.

**Notes** — the torso and legs are both recorded when the trio is playing, which means a
frame contributes two animation counts. The counts are therefore not a partition of frames;
they are per-channel occupancy, and the report's rows are not comparable across channels.

## `show_animation_stats`

**Contract** — print both reports, animations first and transitions second, each sorted
ascending by count. Prints nothing if nothing has been recorded. Invoked from the console.

**Notes** — ascending order puts the **never-played and rarely-played entries at the top**,
which is the question being asked: a motion the engine cannot reach is either an unused
authoring slot or a selection bug, and both are found at the low end. A report sorted the
usual way would bury them.

The sort is done over an array of pointers into the tables rather than over the tables, so
the keyed structures are not disturbed and the report can be run repeatedly while the game
continues.

## Could not recover

Each animation entry carries the visual name it was first seen on. Nothing reads it, and no
report column shows it — the field is constructed and ignored. It was presumably meant to
attribute a motion to the model that used it, which would make the report usable on a level
with several stalker models; as it stands the report merges them.
