# src/xrGame/stalker_velocity_collection.cpp

> The nineteen movement speeds of one kind of stalker, read out of configuration once.

**Needs** — [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md)
**Used by** — [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md)
**Tier floor** — T2: a fixed table loaded from `ltx`.

## Purpose

A stalker's walking speed is not one number. It depends on its **mental state** (free,
danger, panic), its **body state** (standing, crouching), its **movement type** (walk, run)
and its **movement direction** (forward, back, left, right). This file loads that whole
table for one configuration section, so that the movement layer can answer "how fast am I
going" by indexing rather than by re-reading configuration every time the creature's posture
changes — which, in combat, is several times a second.

Why it is a separate object rather than fields on the creature: many stalkers share one
section, and there are a few dozen sections against a few hundred creatures. The table is
loaded once per section and shared; see
[`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md).

## State

The table is not a uniform four-dimensional array, and its sparseness *is* the behaviour
model.

```text
RECORD StalkerVelocityCollection
  danger : real[2][2][4]   # body state x movement type x direction — sixteen entries
  free   : real[2]         # movement type only
  panic  : real            # one number
```

**Invariants** — three shapes, because three mental states move differently:

- **Danger** is the only state with the full cross product. A stalker that expects to be
  shot at moves in all four directions, standing or crouched, walking or running: it
  strafes, it backs away, it advances crouched. Sixteen tuned numbers.
- **Free** is always standing and always forward. An unworried stalker walking to work does
  not sidestep. Two numbers — walk and run.
- **Panic** is standing, running, forward. One number. A fleeing stalker has exactly one
  way of moving.

Every entry is a required key in the section; there is no default and no fallback. A section
missing one of them fails to load, which is the right call — a missing speed would silently
become zero and the creature would freeze in one specific posture, a bug that is nearly
impossible to see.

## `construct(section)`

**Contract** — reads all nineteen values from one configuration section and fills the table.
Blocks on the configuration layer. Fails if any key is absent.

```text
FUNCTION construct(section)
  FOR EACH body IN [crouch, stand]
    FOR EACH type IN [walk, run]
      FOR EACH dir IN [forward, backward, left, right]
        danger[body][type][dir] := config.real(section, "danger_" + body + "_" + type + "_" + dir)
  free[walk] := config.real(section, "free_stand_walk_forward")
  free[run]  := config.real(section, "free_stand_run_forward")
  panic      := config.real(section, "panic_stand_run_forward")
```

**Notes** — the key names encode the full posture even where the table does not: the free
and panic keys spell out `stand` and `forward` although nothing else is possible. Keeping
the naming uniform is what lets a designer read the section as a flat list of postures. The
key names are part of the shipped game data and are therefore frozen.

The reading loop in the original is written out longhand, sixteen calls with sixteen
literal key names, rather than composed from the enumeration names. A rebuild should
compose them — but must reproduce the exact spellings above, because the data files carry
them.
