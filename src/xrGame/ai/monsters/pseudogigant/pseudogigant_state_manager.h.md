# src/xrGame/ai/monsters/pseudogigant/pseudogigant_state_manager.h

> Declares the pseudogiant's brain: nine shared states and an overriding selector, with nothing giant-specific in either.

**Needs** — [`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md)
**Used by** — [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) · [`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md)
**Tier floor** — T3: a priority selector over a registered state set

## Purpose

Declares the surface implemented in
[`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md). Nothing here is
overridable by a further derived brain — no creature derives from the giant — so the only
reason `execute` is virtual is that the base declares it so.

## Exported units

- construction — registers the nine states.
- `execute` — the selector.
- `remove_links` — forwards to the base cascade.
