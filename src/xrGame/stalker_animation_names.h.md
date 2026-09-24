# src/xrGame/stalker_animation_names.h

> The name-fragment tables from which every stalker animation identifier is assembled, plus the critical-wound taxonomy.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`stalker_animation_data.cpp`](stalker_animation_data.cpp.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`stalker_animation_names.cpp`](stalker_animation_names.cpp.md) · [`stalker_animation_state.cpp`](stalker_animation_state.cpp.md) · [`stalker_animation_state.h`](stalker_animation_state.h.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T3: a table of text fragments and an enumeration

## Purpose

A stalker's animation set is not enumerated anywhere in the engine. It is *computed*: a
base visual name is concatenated with one fragment from each of several tables, and the
resulting string is looked up in the model's motion bank. This header declares those
tables. The substance — which fragments exist, in which order, and what each index means —
is in [`stalker_animation_names.cpp`](stalker_animation_names.cpp.md), because the index
of a fragment within its table *is* the value the rest of the AI code indexes by.

The critical-wound enumeration lives here because its values are literally indices into
one of those tables.

## `ECriticalWoundType`

**Contract** — names the six body regions a critical wound can land on: head, torso, left
hand, right hand, left leg, right leg, plus a "none" value.

**Invariants** — the first member starts at **4**, not 0, and the six members are
consecutive. That offset is not arbitrary: it makes the enumeration a direct index into
the global animation-name table, where entries 0–3 are non-wound animations and entries
4–9 are the first severity tier of critical hits. A rebuild that renumbers this
enumeration must renumber that table in lockstep, or decouple the two with an explicit
mapping.

The "none" value is the all-ones pattern of an unsigned 32-bit word, which is this
codebase's universal "no value" for an identifier. A rebuild with an optional type should
use that instead.

## Exported tables

Ten tables are declared; eight are defined and two are not — see
[`stalker_animation_names.cpp`](stalker_animation_names.cpp.md) for what each contains and
which pair is dead.
