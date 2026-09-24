# src/editors/xrWeatherEditor/property_vec3f_reference.cpp

> A grid row bound directly to a three-component vector the engine owns — the smallest place the two-runtime boundary shows itself.

**Needs** — [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md) · [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md)
**Tier floor** — T1: a user-interface object holds a raw reference into engine memory, with a hand-written release.

## Purpose

Eight lines of code and one whole problem. The grid's rows live in the user-interface
runtime, which reclaims objects on its own schedule; the value this row edits lives in the
engine's memory, which does not. A row that points straight at engine memory therefore
needs a **release step the interface runtime will honour**, and that is what this file is.

## State

See [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md).

## The boundary problem, stated once

```text
# the row lives on the interface side; the value lives on the engine side
FUNCTION construct(target : reference to vec3)
  binding = allocate_on_the_engine_side(holder_of(target))
  # the base row is also told the initial value, for its display

FUNCTION release()                 # runs when the interface reclaims this row,
  free(binding)                    # deterministically, before the reclaim completes

FUNCTION read()  -> vec3 : RETURN binding.value
FUNCTION write(v : vec3)  : binding.value = v
```

**Contract** — reading and writing go straight through to the engine's field; there is no
copy and no notification. Releasing frees the binding exactly once.

**Invariants** — the referenced vector must outlive the row, and nothing checks it. In
practice the engine record and the row are created and destroyed together by the same
`fill` and teardown pair in the weather model, which is the only reason this holds.

**Notes** — This is the whole managed-to-native ownership question in miniature, and a
rebuild should read it as a **requirement, not as a technique**:

- Every object on the interface side that points into engine memory must have a release
  that runs at a *defined moment*, not whenever the interface runtime gets around to it.
  The original spells this out with two separate destructors — one for the deterministic
  path, one for the collected path, the first delegating to the second — which is the
  platform's idiom for exactly this and is incidental.
- The binding is allocated on the engine's side of the boundary, not the interface's,
  because the interface runtime may relocate what it owns and the engine holds the address.
- The direction of trust is one-way: **the row trusts the engine's field to outlive it;
  the engine never learns that a row exists.** Every other row type in this family repeats
  the arrangement.

A rebuild in a single runtime deletes this file and the dozen like it, and keeps the
lifetime rule: the grid's view of a value is valid only while the model object behind it
lives, and the model's teardown must therefore tear the grid down first — which is what
[`ide.hpp`](../xrWeatherEngine/ide.hpp.md) on the engine side is arranging.
