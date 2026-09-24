# src/xrGame/ai_space.h

> Declares the AI layer's global service holder and the short name every caller reaches it by.

**Needs** — [`ai_space.cpp`](ai_space.cpp.md) · [`ai_space_inline.h`](ai_space_inline.h.md) · [`xrAICore/AISpaceBase.hpp`](../xrAICore/AISpaceBase.hpp.md) · [`xrCore/Events/Notifier.h`](../xrCore/Events/Notifier.h.md)
**Used by** — [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`GameObject.cpp`](GameObject.cpp.md) · [`GameTask.cpp`](GameTask.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`Level_load.cpp`](Level_load.cpp.md) · [`Level_network_messages.cpp`](Level_network_messages.cpp.md) · [`Level_network_spawn.cpp`](Level_network_spawn.cpp.md) · [`Level_network_start_client.cpp`](Level_network_start_client.cpp.md) · [`MainMenu.cpp`](MainMenu.cpp.md) · [`MainMenu.h`](MainMenu.h.md) · [`abstract_location_selector_inline.h`](abstract_location_selector_inline.h.md) · [`abstract_path_manager_inline.h`](abstract_path_manager_inline.h.md) · [`action_planner_inline.h`](action_planner_inline.h.md) · _and 94 more_
**Tier floor** — T3: a declaration.

## Purpose

Declares the surface implemented in [`ai_space.cpp`](ai_space.cpp.md) and
[`ai_space_inline.h`](ai_space_inline.h.md). Extends the AI core's own space base — which
owns the navigation graphs and the path engine — with the game layer's services.

Exported units:

- **`GetInstance`** and the free name **`ai()`** — the process-global instance.
- **Event subscription** — `Subscribe`, `Unsubscribe`, over two events:
  *script engine started* and *script engine reset*.
- **`RestartScriptEngine`** — the only public way to cycle the script virtual machine.
- **Service accessors** — `ef_storage`, `alife` and `get_alife`, `cover_manager`,
  `get_moving_objects`, `doors`.

**Notes** — The level lifecycle (`init`, `load`, `unload`, `set_alife`) is private and
opened only to a named list of friends: the off-screen simulation and three of its
registries, plus the level. That list is the honest statement of who is allowed to move
the AI space between states, and a rebuild should express it as a narrow interface rather
than as friendship.

Copying is forbidden, which is the type asserting it is a singleton by construction rather
than by convention.
