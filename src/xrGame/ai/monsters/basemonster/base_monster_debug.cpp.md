# src/xrGame/ai/monsters/basemonster/base_monster_debug.cpp

> The creature's introspection: a full tree of everything it currently knows, believes and is doing, built only in a development build.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`state_manager.h`](../state_manager.h.md) · [`control_manager.h`](../control_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`ai_monster_squad.h`](../ai_monster_squad.h.md) · [`level_debug.h`](../../../level_debug.h.md) · [Seam: Debug overlay UI](../../../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: string formatting into a tree

## Purpose

Entirely absent from a shipping build. A rebuild may skip it completely and lose nothing the
player can see.

It is worth a page anyway, for one reason: **the tree it builds is the authoritative list of
what constitutes a creature's mental state**. The original has no design document; this
routine is the closest thing to one, because it was written by someone who needed to see
everything that could be wrong.

## What the tree contains

```text
General
  name, class, section, visual, position, navigation vertex
  health, morale, and the four mood flags (angry, growling, aggressive, asleep)
  Perceptors
    Visual   — effective eye range and field of view after per-state modulation,
               and whether the player is visible right now
    Sounds   — how many sounds are remembered, plus the most recent and the oldest,
               each with its source, position, type, power and whether it was dangerous
    Hits     — how many hits are remembered, plus the last one's source, time,
               position and direction
  Corpse manager — the corpse currently chosen, and satiety
  Group behaviour — team, squad and group identifiers; whether the pack is active,
               whether I lead it, who does, how many are alive, and both my
               current command and my reported goal with its subject
Brain
  Fsm — the state machine's own tree: the current state and its substates
  Script control — the controlling script's name, the current action and the next
  Control manager — which component owns each channel
  Map home — the home territory's three radii
  ... and, per creature kind, whatever the concrete creature adds
```

**Notes** — the *effective* eye range and field of view are reported rather than the
configured ones, because both are modulated by the creature's current state (a resting
creature sees less than an alerted one). That modulation is invisible anywhere else, which is
why it is reported here.

The pack section reports both directions of the pack protocol — the command handed down and
the goal reported up — side by side. That pairing is the quickest way to see a coordination
bug, and it is why the two vocabularies in
[`ai_monster_squad.h`](../ai_monster_squad.h.md) have human-readable names at all.

The living-member count is reported as one when the pack reports zero and the creature is
alive, compensating for the pack's deliberate conflation of "one member" with "no pack".

## `show_debug_info`

**Contract** — draws a compact two-column overlay for the creature the developer has
selected, and returns a descriptor saying where the next block should be drawn. Selection is
per creature and has three levels: off, first column, second column.

## `debug_state_machine`

**Contract** — draws the state machine's current state and its history on screen, driven by
a global AI debug flag.

## the value formatters

**Contract** — three small routines turning a sound's danger grade, a pack goal kind and a
pack command kind into readable text. They exist because those three enumerations are the
ones a developer reads most often.

## the debug variable table

**Contract** — referenced from [`base_monster.h`](base_monster.h.md) rather than defined
here: a creature can look up any tuned parameter by the name
`<creature class name>_<parameter name>` in a global table a developer edits at runtime, and
use the override in place of the configured value. In a shipping build the lookup compiles
away and the configured value is used directly.

**Notes** — this is how the attack-on-move parameters were tuned, and it is the reason those
accessors are functions rather than field reads. A rebuild that has a live configuration
reload does not need it.
