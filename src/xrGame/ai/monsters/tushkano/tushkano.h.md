# src/xrGame/ai/monsters/tushkano/tushkano.h

> Declares the tushkano — the small scavenging rodent, the simplest creature in the game.

**Needs** — [`basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`controlled_entity.h`](../controlled_entity.h.md) · [`tushkano.cpp`](tushkano.cpp.md)
**Used by** — [`tushkano.cpp`](tushkano.cpp.md) · [`tushkano_script.cpp`](tushkano_script.cpp.md) · [`tushkano_state_manager.cpp`](tushkano_state_manager.cpp.md)
**Tier floor** — T3: a class identity and one animation table

## Purpose

Declares the surface implemented in [`tushkano.cpp`](tushkano.cpp.md). The tushkano adds
almost nothing to the generic creature: no abilities, no special senses, no custom hit
handling. What it is, it is by virtue of its animation table and its configuration section.

It does compose in the *controllable* mixin, which every creature that a controller
psychic can enslave must carry. That is the only capability it has beyond the base.

## Exported units

- **construction** — installs the tushkano's behaviour tree and registers itself as
  controllable.
- **load** — builds the animation table from the configuration section.
- **the special-parameter hook** — the base class's "the current state asked for an
  animation modifier" callback. The tushkano's is empty (see [`tushkano.cpp`](tushkano.cpp.md)).
- **the class name** — the string the engine uses to identify this creature kind in
  diagnostics and script.
