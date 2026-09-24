# src/xrGame/ik_calculate_data.cpp

> Initializes one limb's solve working set.

**Needs** — [`ik_calculate_data.h`](ik_calculate_data.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: initialization

## Purpose

The construction of the record described in
[`ik_calculate_data.h`](ik_calculate_data.h.md), which holds the substance.

## State

`Stateless.`

## `SCalculateData` (construction)

**Contract** — binds the record to a limb and to the owning object's world transform, and
zeroes everything else: no joint angles, no collision requested, no shift, not to be applied,
both blend fractions zero, and the persistent state default-initialized.

**Invariants** — the object transform is held by reference, so the caller must keep it alive
and unchanged for the duration of the solve. See the invariant in the header's twin: a solve
is expressed against one snapshot of the owner's position.

The persistent state is default-constructed here, which means **a fresh working set does not
carry the limb's previous frame's state**. The caller is responsible for copying the limb's
saved state in; this constructor is used where a solve starts from nothing.
