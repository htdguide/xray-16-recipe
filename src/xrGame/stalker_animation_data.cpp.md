# src/xrGame/stalker_animation_data.cpp

> Loads one stalker skeleton's entire animation table by name from three prefix families, once per distinct model.

**Needs** — [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`stalker_animation_state.h`](stalker_animation_state.h.md) · [`stalker_animation_names.h`](stalker_animation_names.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — reached through its declarations in [`stalker_animation_data.h`](stalker_animation_data.h.md); callers name that, not this file.
**Tier floor** — T3: three table loads by generated name

## Purpose

Every motion a stalker can play is found once, up front, by composing names from fixed word
lists and looking each one up in the model's motion bank. After this constructor runs, every
animation selection anywhere in the stalker's animation system is an array index — no string
lookup happens at run time.

## State

```text
RECORD StalkerAnimationData
  part_animations   : table indexed by body state, each holding the torso, movement,
                      in-place and global motions for that state
  head_animations   : flat table of head motions
  global_animations : table indexed by weapon kind, each holding that weapon's
                      whole-body actions
```

## Constructor

**Contract** — load all three tables from one animated skeleton. The body-state and head
tables are loaded with no name prefix; the whole-body weapon-action table is loaded with the
prefix `item_`. Blocks; runs once per distinct model, not once per stalker.

**Invariants** — the prefix is the only difference between the three loads, and it is
load-bearing: the whole-body weapon actions are authored under `item_`-prefixed names in the
shipped model files, and the other two families are not. A rebuild must generate exactly
these names or the lookups miss and the stalker has no animations.

**Notes** — the structure of each table — which axes it is indexed by, in what order, and
which word list supplies each axis's names — is in
[`stalker_animation_state.cpp`](stalker_animation_state.cpp.md) and
[`stalker_animation_names.cpp`](stalker_animation_names.cpp.md). This file only says that
all three are loaded together, which is what makes them shareable as a unit.
