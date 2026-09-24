# src/xrGame/player_account.h

> Declares the online identity a multiplayer player carries into a match — nickname, clan, profile number — implemented in [`player_account.cpp`](player_account.cpp.md).

**Needs** — [`profile_data_types.h`](profile_data_types.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`game_base.cpp`](game_base.cpp.md) · [`game_base.h`](game_base.h.md) · [`player_account.cpp`](player_account.cpp.md)
**Tier floor** — T2: a declaration with a frozen wire layout

## Purpose

Declares `player_account`: the part of a multiplayer player that comes from the matchmaking
service rather than from the game. It is attached to a player and transmitted with them so
every client can display who everyone is.

One property here is worth naming before the implementation: **the account distinguishes
online from offline identity**, and the distinction is enforced rather than informational.
A player signed in to the account service has a name the service owns and the game may not
change; a player who is not signed in has a locally chosen name. That is the reason the
name setter exists at all and the reason it refuses in one of the two cases.

Exported units:

- `player_account` — the identity.
- `name` / `clan_name` / `profile_id` / `is_clan_leader` / `is_online` — the five readable
  properties.
- `load_account` — fill from the locally signed-in profile.
- `net_Import` / `net_Export` — the wire form.
- `skip_Import` — consume an account from a message without keeping it.
- `set_player_name` — rename, permitted only for an offline identity.
