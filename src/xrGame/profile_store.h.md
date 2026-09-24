# src/xrGame/profile_store.h

> Declares the player-profile store: the holder of a signed-in player's awards and best scores.

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [`profile_data_types_script.h`](profile_data_types_script.h.md)
**Used by** — [`MainMenu.cpp`](MainMenu.cpp.md) · [`account_manager_console.cpp`](account_manager_console.cpp.md) · [`player_account.cpp`](player_account.cpp.md) · [`profile_store.cpp`](profile_store.cpp.md) · [`profile_store_script.cpp`](profile_store_script.cpp.md) · [`ui_export_script.cpp`](ui_export_script.cpp.md)
**Tier floor** — T3: two container fields and an asynchronous load

## Purpose

Declares the surface implemented in [`profile_store.cpp`](profile_store.cpp.md) and
exported in [`profile_store_script.cpp`](profile_store_script.cpp.md).

Exported units:

- `load_current_profile(progress_cb, complete_cb)` — asynchronously fetch the signed-in
  player's awards and best scores; report through the completion callback.
- `stop_loading()` — cancel an in-flight load. Does nothing: there is nothing to cancel.
- `get_awards()` · `get_best_scores()` — the two sets, handed to scripts as iterators.

**Notes** — both sets are marked in the original as *dummies*. They are constructed empty
and never written, because the service that would fill them is gone. They exist so the
shipped profile screen finds the methods it calls and renders an empty profile rather than
failing. See [Seam: Multiplayer matchmaking and
accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts).
