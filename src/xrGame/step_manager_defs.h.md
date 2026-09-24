# src/xrGame/step_manager_defs.h

> The two records footsteps are described with: the authored schedule and the live per-clip
> state.

**Needs** — [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`step_manager.cpp`](step_manager.cpp.md) · [`step_manager.h`](step_manager.h.md)
**Tier floor** — T2: two small records and an enumeration.

## Purpose

The data definitions behind [`step_manager.cpp`](step_manager.cpp.md), separate because
several creature classes name these types in their own declarations without needing the
manager's implementation. The split is organisational and a rebuild may merge it; what is
substantive is the shape of the two records, which is the footstep model itself.

## State

```text
CONSTANT min_legs = 1
CONSTANT max_legs = 4

ENUM LegType = front_left | front_right | back_right | back_left

RECORD StepParams                              # authored, one per animation
  step   : { time : real, power : real }[max_legs]
  cycles : int (8-bit)

RECORD StepInfo                                # live, for the clip now playing
  activity  : { handled : bool, cycle : int (8-bit) }[max_legs]
  params    : StepParams
  disable   : bool                             # initially true
  cur_cycle : int (8-bit)
```

**Invariants**

- `step[leg].time` is a **fraction of one cycle**, not of the clip, so a schedule authored
  once serves a clip of any length that declares how many cycles it contains. This is the
  single most important line in the file.
- `power` is the footfall's loudness, per leg — a limp is authored by giving one leg less
  power, not by a separate animation.
- `cycles` is at least one and is checked at load; zero would make the cycle duration
  infinite.
- The leg enumeration's **order is the configuration format's order**. A schedule line lists
  its time/power pairs in this sequence, so renaming or reordering the enumeration silently
  reassigns every authored footfall in the shipped game data. It is frozen.
- A quadruped's ordering — both front legs, then back right, then back left — is not a
  rotation and not left-to-right. It is what the data uses.
- `disable` starts true, so a manager that has not yet been given a schedule is silent
  rather than firing against an empty one.
- `activity[leg].cycle` records *which* cycle a leg last fired in, not merely that it fired.
  A plain flag would fire each leg once per clip instead of once per cycle.

## Notes

Both arrays are sized for the maximum leg count rather than for the creature's actual one, so
a biped carries two unused entries. That is the right trade for a record instantiated once
per creature and walked every frame: a fixed-size array keeps it one allocation-free block.
A rebuild may size it exactly; nothing depends on the padding.

The eight-bit widths are incidental compaction, not a wire format — nothing here is
serialized.
