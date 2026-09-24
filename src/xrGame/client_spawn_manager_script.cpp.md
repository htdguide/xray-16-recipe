# src/xrGame/client_spawn_manager_script.cpp

> Exports the spawn-notification registry to the script virtual machine.

**Needs** — [`client_spawn_manager.h`](client_spawn_manager.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point makes the registry described in
[`client_spawn_manager.cpp`](client_spawn_manager.cpp.md) usable from Lua. It is how a
script says "when entity 4271 exists, call me" — the standard way a script acts on an
identifier read out of a save, a dialogue or a smart terrain's job assignment without
polling.

## State

`Stateless.`

## `CClientSpawnManager::script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up.
The type is named `client_spawn_manager` to scripts, with three methods:

- **`add(awaited_id, waiter_id, function, receiver)`** — register a callback with a bound
  receiver, which is how a script registers a method on one of its own tables.
- **`add(awaited_id, waiter_id, function)`** — register a plain function.
- **`remove(awaited_id, waiter_id)`** — deregister.

Names and signatures are frozen by conformance criterion 10.

**Notes** — the engine-side registration overload and both bulk-clear operations are
deliberately not exported. A script may only manage its own waits, one at a time; it cannot
clear the registry or register a native callback.

**Notes** — the argument naming follows the implementation's, which the reverse lookup
contradicts — see [`client_spawn_manager.cpp`](client_spawn_manager.cpp.md). Scripts pass
the awaited identifier first.
