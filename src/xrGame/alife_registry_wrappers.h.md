# src/xrGame/alife_registry_wrappers.h

> Names one wrapper type per persistent registry, so a game object can hold "my news feed" or "my relations" as a field.

**Needs** — [`alife_registry_container.h`](alife_registry_container.h.md) · [`alife_registry_wrapper.h`](alife_registry_wrapper.h.md)
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`LevelFogOfWar.cpp`](LevelFogOfWar.cpp.md) · [`PDA.cpp`](PDA.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`inventory_owner_info.cpp`](inventory_owner_info.cpp.md) · [`map_manager.cpp`](map_manager.cpp.md) · [`relation_registry.cpp`](relation_registry.cpp.md) · [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) · [`UILogsWnd.cpp`](ui/UILogsWnd.cpp.md) · [`UITalkDialogWnd.cpp`](ui/UITalkDialogWnd.cpp.md)
**Tier floor** — T3: declarations only

## Purpose

A game object that needs persistent side-state embeds one of these. Each is a
per-owner handle onto one registry (see
[`alife_registry_wrapper.h`](alife_registry_wrapper.h.md)) wrapped once more so that the
handle itself is heap-allocated and reached through an accessor.

That extra wrapping exists for a single reason: **the handle must be constructible before
the owner's identifier is known**, and destructible independently of the alife simulation.
Giving every consumer a separately allocated handle also means the consumer's own header
need not know the registry's value type, which is what keeps the registry definitions out
of a dozen unrelated headers. A rebuild in a language without header-compilation costs
should embed the handle directly and delete this layer.

## State

Stateless as a file. It names seven consumer-facing handle types, one per registry that
a game object holds directly:

```text
known contacts      # the player's conversation partners
encyclopedia        # the player's unlocked articles
game news           # the player's news feed
known information   # a character's learned information portions
character relations # a character's personal goodwill toward others
map locations       # the player's map markers
game tasks          # the player's tasks
actor statistics    # the player's tallies
```

Two registries from the container's list have no handle here — the specific-character
registry and, historically, a fog-of-war registry that has been removed. The first is
consulted by the character-creation code directly rather than held by any object; the
second is gone from the container list entirely, and its absence is part of the current
save format.

## The wrapping type

**Contract** — owns exactly one per-owner registry handle, created when the consumer is
created and released when it is released, and exposes it by reference. The handle is
required to exist for the consumer's whole lifetime.

**Notes** — the allocation is unconditional and eager: every character carries a handle
for every registry it declares, whether or not it will ever use one. With a few thousand
characters that is a few thousand small allocations, which is the kind of thing the
engine's pooled allocator exists to absorb. A rebuild embedding the handle by value
removes the cost entirely.
