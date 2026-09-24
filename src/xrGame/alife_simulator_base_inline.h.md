# src/xrGame/alife_simulator_base_inline.h

> The accessors for the alife simulation's eleven sub-objects, each guarded by the same initialization check.

**Needs** — [`alife_simulator_base.h`](alife_simulator_base.h.md)
**Used by** — [`alife_simulator_base.h`](alife_simulator_base.h.md)
**Tier floor** — T3: field reads behind one guard

## Purpose

Bodies for the simulator base's accessors. Substance is in
[`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) and
[`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md).

Almost nothing here is a decision. The one thing that is, is repeated twenty-six times
and is worth stating once.

## The guarded accessor shape

**Contract** — every sub-object accessor asserts two things before yielding a reference:
that the simulation is **initialized**, and that the particular sub-object exists. It then
returns the sub-object by reference; there is no failure path and no absent result.

**Invariants** — this is how the alife layer expresses "the simulation either exists
completely or does not exist at all". Callers never test for a missing registry; they test
for a missing *simulation* — through a separate global query — and once past that, every
part of it is present. A rebuild should make the simulation an optional whole rather than
eleven optional parts, and then delete every one of these checks.

The accessors come in readable and writable pairs so that code holding an immutable view
of the simulation cannot mutate a registry through it. That distinction is real and worth
keeping; which callers get which is decided by the protected/public split described in
[`alife_simulator_base.h`](alife_simulator_base.h.md).

## The accessors

**Contract** — header, time manager (under two names), spawn registry, object registry,
graph registry, schedule registry, story registry, smart-terrain registry, group registry,
persistent registry container, inventory-upgrade manager. Plus:

- `random` — the simulation's own random stream, ungarded because it is a value member.
- `server` — the network server; asserted present, but not tied to initialization, because
  it is supplied at construction and outlives the built state.
- `server_command_line` — the session string the simulator rewrites at startup.
- `setup_command_line` — records that string by reference.

## `can_register_objects`

**Contract** — sets the gate that suppresses entity registration hooks during a bulk
load. **Asserts that the value actually changes**, which makes the gate strictly a
matched pair: whoever lowers it must raise it, exactly once, and nesting is forbidden.

**Notes** — that assertion is the only thing preventing the gate from being left down. A
rebuild expressing it as a scoped bracket around the load gets the same guarantee
structurally, and should.
