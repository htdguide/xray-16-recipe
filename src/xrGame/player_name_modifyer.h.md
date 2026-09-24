# src/xrGame/player_name_modifyer.h

> Declares the one function that makes a player-chosen nickname safe to put in a file path and a console line.

**Needs** — [`player_name_modifyer.cpp`](player_name_modifyer.cpp.md)
**Used by** — [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`player_name_modifyer.cpp`](player_name_modifyer.cpp.md)
**Tier floor** — T4: one declaration

## Purpose

Declares `modify_player_name`, implemented in
[`player_name_modifyer.cpp`](player_name_modifyer.cpp.md). A free function with no state and
no type around it, because sanitizing a nickname is not anybody's responsibility in
particular.
