# src/xrGame/ef_storage.cpp

> Builds every evaluation function at startup, assigns each primary one a fixed numeric identity, and loads the twenty-four trained functions from data files.

**Needs** — [`ef_storage.h`](ef_storage.h.md) · [`ef_primary.h`](ef_primary.h.md) · [`ef_pattern.h`](ef_pattern.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — reached through its declarations in [`ef_storage.h`](ef_storage.h.md); callers name that, not this file.
**Tier floor** — T3: construction and a linear name lookup

## Purpose

The one place the evaluation-function catalogue is written down. Everything about it that a
rebuild must reproduce exactly is in the *numbers*: which function occupies which slot, and
which data file supplies which trained function.

## State

```text
RECORD Storage
  base_functions : list<optional<BaseFunction>>    # exactly 128 slots, mostly empty
  ... plus a named handle for each function, aliasing the same objects
  non_alife_params, alife_params : Params          # see ef_storage.h
```

**Invariant** — the slot index **is** the function's identity in the trained data. A trained
function's data file lists the slot numbers of the primary functions it takes as inputs, so
renumbering a slot silently rewires every trained function that referenced it. The shipped
data files are frozen, so the numbering is frozen.

**Invariant** — the slot table is the ownership. The named handles alias the same objects
and destruction walks the table, so a function that is *only* reachable through a named
handle is leaked. Every trained function is in exactly that position — they are not placed in
the table by the constructor but by the loader; see below.

## The slot map

Three blocks with deliberate gaps between them, and the gaps are reserved space:

| Slots | Block | Functions |
|---|---|---|
| 0–9 | **item** properties | distance, graph-point type, equipment type, item deterioration, equipment preference, main-weapon type, main-weapon preference, item value, weapon ammunition count, detector type |
| 10–20 | *unused* | reserved |
| 21–31 | **personal** properties of the evaluating creature | health, morale, creature type, weapon type, accuracy, intelligence, relation, greed, aggressiveness, eye range, maximum health |
| 32–40 | *unused* | reserved |
| 41–50 | **enemy** properties | health, creature type, weapon type, equipment cost, rucksack weight, anomality, eye range, maximum health, anomaly type, distance to graph point |
| 51–127 | trained functions, filled at load | see below |

**Invariants** — the three ten-or-eleven-slot blocks at multiples of roughly twenty are a
numbering convention with room to grow, not an accident. Five of the enemy functions are the
[enemy-perspective wrapper](ef_storage.h.md) applied to the matching personal function rather
than distinct code.

## The trained functions

**Contract** — twenty-four pattern functions, each constructed from a named data file under
the AI data root, in two directories:

- **`common`** (twelve): weapon effectiveness, creature effectiveness, intellect-creature
  effectiveness, accuracy-weapon effectiveness, final creature effectiveness, victory
  probability, entity cost, expediency, surge death probability, equipment value, main-weapon
  value, small-weapon value.
- **`alife`** (twelve): terrain type, weapon attack times, weapon success probability, enemy
  detectability, enemy detect probability, enemy retreat probability, anomaly detect
  probability, anomaly interact probability, anomaly retreat probability, birth percentage,
  birth probability, birth speed.

**Invariants** — the split names the two consumers. The `common` set is used by both
simulations; the `alife` set is the offline simulation's own — it decides whether creatures
breed, whether they find each other, whether they run from anomalies. That is a substantial
part of what makes the world feel alive while nobody is watching, and it is all table lookup
against data fitted offline.

**Invariants** — each file's *contents* say which slot the function occupies (see
[`ef_pattern.cpp`](ef_pattern.cpp.md)), so the constructor here does not assign one. The
consequence is that a data file with a wrong function-type field overwrites another
function's slot, and nothing detects it.

**Invariants** — every file is required. A missing one is a hard failure at startup, not a
degraded mode.

## `function`

**Contract** — finds a function by name with a linear scan of the slot table, skipping empty
slots; returns nothing on a miss. Names are compared exactly. Used only by the script
surface, where a miss is reported as a script error rather than a failure.

**Notes** — a linear scan over 128 slots per script call is unremarkable at the rate scripts
evaluate, but the name is a fixed string in a fixed-size buffer, which is a C++ artifact. A
rebuild uses a map.

## Destruction

**Contract** — deletes every occupied slot. See the ownership invariant above: a trained
function that never got into the table is not freed.
