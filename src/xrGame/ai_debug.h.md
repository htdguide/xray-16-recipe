# src/xrGame/ai_debug.h

> The AI layer's debug switch set: one bit per diagnostic, in one flag word the whole game module reads.

**Needs** — _(none)_
**Used by** — [`action_planner.h`](action_planner.h.md) · [`ai_monsters_anims.h`](ai/ai_monsters_anims.h.md) · [`monster_state_rest_fun.h`](ai/monsters/states/monster_state_rest_fun.h.md) · [`monster_state_rest_sleep.h`](ai/monsters/states/monster_state_rest_sleep.h.md) · [`alife_level_registry.h`](alife_level_registry.h.md) · [`alife_object_registry.cpp`](alife_object_registry.cpp.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`cover_evaluators.cpp`](cover_evaluators.cpp.md) · [`ef_pattern.cpp`](ef_pattern.cpp.md) · [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`script_action_planner_wrapper.cpp`](script_action_planner_wrapper.cpp.md)
**Tier floor** — T3: a set of named bits.

## Purpose

Turning AI diagnostics on and off is done by bit, not by a per-subsystem variable, because
the checks appear in inner loops and a single word in cache is the cheapest test. The
names below are the vocabulary of the engine's AI debug console and appear in shipped
developer configuration, so they are part of the frozen console surface even though the
behaviour behind them is developer-only.

## State

```text
RECORD AiDebugFlags   # one flag word, bit positions fixed
  # available only in a checked build:
  0  Debug                 general AI tracing
  1  Brain                 the per-creature decision loop
  2  Motion                movement selection
  3  Frustum               vision frustum drawing
  4  Funcs                 function-level tracing
  7  GOAP                  the planner's search
  8  Cover                 cover selection
  9  Animation             animation selection
  10 Vision                what is seen and why
  11 MonsterDebug
  12 Stats
  13 Destroy               entity teardown ordering
  14 Serialize             save/load of AI state
  15 Dialogs
  16 InfoPortion           knowledge flags handed between characters
  17 GOAPScript            script-added planner elements
  18 GOAPObject            planner elements attached to objects
  19 Stalker
  20 DrawGameGraph         overlay the cross-level graph
  21 DrawGameGraphStalkers
  22 DrawGameGraphObjects
  25 DebugOnFrameAllocs    catch per-frame allocation
  26 DrawVisibilityRays
  27 AnimationStats
  28 DrawGameGraphRealPos

  # available in any build that is not the shipping gold build:
  5  ALife                 the off-screen simulation's tracing
  24 IgnoreActor           creatures do not perceive the player
  28 ObstaclesAvoiding
  29 ObstaclesAvoidingStatic
  30 UseSmartCovers
  31 UseSmartCoversAnimationSlot
```

**Invariants** — Bit 28 is claimed twice, once by each group, and the two groups are
compiled together in a checked build. This is a real collision: in a checked non-gold
build, drawing real positions on the game graph and avoiding obstacles are the same
switch. It looks like an oversight rather than a decision, and a rebuild should renumber.

**Notes** — Bits 6 and 23 are unused; 23 is explicitly vacated by a note saying the
nil-object-access diagnostic it named belongs in the script engine instead. The gaps mean
the numbering is not free to be compacted — developer configuration files set these by
value.

The split between the two groups is the load-bearing part: the *simulation* switches
(off-screen simulation tracing, actor perception, obstacle avoidance, smart cover use)
survive into a non-gold release build, because they are the ones needed to diagnose a
report from a player build. Pure drawing and tracing do not.
