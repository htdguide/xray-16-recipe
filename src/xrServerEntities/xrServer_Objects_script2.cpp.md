# src/xrServerEntities/xrServer_Objects_script2.cpp

> Exports the physics-skeleton mixin and the visual record level.

**Needs** — [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The continuation of [`xrServer_Objects_script.cpp`](xrServer_Objects_script.cpp.md), split
for build time rather than for any organizing reason. Two registrations.

## `cse_ph_skeleton`

**Contract** — the ragdoll mixin, registered as an opaque type with no members. A script can
tell that a record has one; it cannot read or write the bone states. Those are large, they
change every tick, and nothing in the script layer has a use for them.

## `CSE_AbstractVisual`

**Contract** — a record that is both abstract and a visual. Exported at the abstract level,
so a script subclass may override the serializations, plus one accessor: the startup
animation name.

**Notes** — **this is the only record exported under a name that is not lowercase.** Every
other type in the chapter uses the `cse_*` convention; this one carries its internal name.
It is a slip, it is frozen by conformance criterion 10, and a rebuild must reproduce the odd
spelling.

The startup animation accessor is the only piece of the visual facet reachable from script,
and it exists because a script record that poses itself needs to know which animation it was
authored with.
