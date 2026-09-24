# src/xrGame/xrGame.cpp

> The game module's entry point: what the engine calls to bring the game layer up, and the two functions through which every entity in the world is created and destroyed.

**Needs** — [`xrGame.h`](xrGame.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../xrUICore/XML/xrUIXmlParser.h.md) · [`xrUICore/ui_styles.h`](../xrUICore/ui_styles.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`xrGame.h`](xrGame.h.md)
**Tier floor** — T1: hands the engine raw function pointers across a dynamic-library boundary and installs an allocator into a third-party library

## Purpose

The engine does not know what a stalker is. It knows that *something* can create an object
from a class identifier and destroy it again, and that something can be asked for a
persistent game state. This file is the entire surface between the two: four operations the
engine calls, and two raw creation functions it is handed pointers to.

Everything else in the chapter hangs off these. A rebuild that fuses the engine and the game
into one program still wants this boundary, because it is where the entity factory's
registration table is guaranteed to exist before the first spawn.

## State

```text
# the module instance itself, one per process, reachable by the engine by name
game_module : GameModule
```

**Invariants** — the module object is constructed before the engine loads it and outlives
every object it creates. Nothing in the game layer may run before `initialize` or after
`finalize`.

## `create_object` (the factory create entry)

**Contract** — given a class identifier, produce a new **client object** of that class.
Consults the object factory's registration table. **Stamps the class identifier onto the
object after construction**, because the constructor does not receive it. Returns nothing
when the identifier is unregistered — but only in checked builds; a release build dereferences
the absent result and crashes, which is the original's deliberate choice that an unregistered
class is a data error caught during development.

**Notes** — the identifier being written *after* construction rather than passed in is the
one real wart. It means an object cannot branch on its own class during construction. The
source flags it as something to fix during factory initialization instead; a rebuild should
pass the identifier to the constructor and delete the stamping.

## `destroy_object` (the factory destroy entry)

**Contract** — destroy an object the create entry produced. The engine never destroys a game
object itself, because the allocator and the destructor both belong to this module.

**Notes** — this pairing exists because in a dynamic-library build the two sides may have
different allocators. Keeping allocation and deallocation on the same side of the boundary
is the problem that solves; a rebuild with one allocator needs the pairing only as a
convention, not as a mechanism.

## `initialize`

**Contract** — bring the game layer up. Called once, by the engine, before anything else in
this module. Hands back the two factory function pointers. Must not touch the graphics device
or the loaded level — neither exists yet.

```text
FUNCTION initialize() -> (create_entry, destroy_entry)
  publish the factory create and destroy entries

  # the alife simulation's clock rate, read here because there is nowhere better
  time_factor := config.real("alife", "time_factor")

  create the user-interface style manager     # before any console command can select a style
  register every console command
  initialize the string table                 # localization; every later message needs it
  register the default input bindings
  route the debug-overlay library's allocations through the engine allocator,
    and bind it to the device's overlay context
  RETURN (create_entry, destroy_entry)
```

**Invariants** — the order is load-bearing in two places. The style manager exists before
console-command registration, because a command can set a style at registration time. The
string table is initialized before anything that might emit a localized message. The rest of
the sequence is independent.

**Notes** — **The alife time factor is read here and the source says so: there is nowhere
better.** It is a single global that the alife simulation, the environment clock and the
script layer all read, and none of them owns it. A rebuild should give it an owner — the
alife simulation is the obvious one — and the only reason not to is that the value must exist
before the simulation does.

Routing the debug-overlay library's allocations through the engine allocator is not
optional: that library allocates during its own shutdown, after the engine's allocator
accounting expects silence, and mismatched allocators across the library boundary is the
failure it avoids. The same problem exists for every third-party library in the build and is
solved the same way.

The input-binding registration is marked as belonging in the engine rather than here. It is
in the game module because the binding *names* are game concepts. A rebuild that keeps
bindings as data rather than as code moves the whole question.

## `finalize`

**Contract** — tear the game layer down. Releases the style manager, the string table and the
input bindings, in that order — the reverse of the parts of `initialize` that own anything.
Called once, after the last object is destroyed.

**Notes** — This is *not* where the game's global registries are released; that is
[`xrgame_dll_detach.cpp`](xrgame_dll_detach.cpp.md), which runs separately and does the
larger teardown. The split is arbitrary and a rebuild should have one teardown.

## `create_persistent` / `destroy_persistent`

**Contract** — produce and release the game layer's persistent state object, the thing that
survives across level loads. Creation also forces the object factory into existence, because
the factory's registration table must be filled before the first spawn and nothing else
guarantees it.

**Notes** — forcing the factory here is a side effect disguised as a no-op call, and the
source flags it for removal. The underlying requirement is real and a rebuild must satisfy it
explicitly: **the class-identifier registration table is complete before any spawn record is
read.**
