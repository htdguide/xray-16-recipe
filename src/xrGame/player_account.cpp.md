# src/xrGame/player_account.cpp

> Who a multiplayer player is according to the account service: their nickname, clan and profile number, and the frozen shape those take on the wire.

**Needs** — [`player_account.h`](player_account.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`login_manager.h`](login_manager.h.md) · [`profile_store.h`](profile_store.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`player_account.h`](player_account.h.md); callers name that, not this file.
**Tier floor** — T1: a message layout that must be read and skipped byte-exactly

## Purpose

A multiplayer player has two identities. The *game* identity — which team, which weapons,
how many kills — belongs to the match. The *account* identity — nickname, clan, profile
number, achievements — belongs to the matchmaking service and is only carried by the match.
This is the second.

The service it draws from no longer exists, so the interesting content of this file is not
the account but **the wire format**, which is frozen against the shipped executable and must
be reproduced exactly by anything that talks to it.

## State

```text
RECORD PlayerAccount
  player_name    : text
  clan_name      : text
  profile_id     : int (32-bit)   # the service's identifier for this account
  clan_leader    : bool
  online_account : bool           # signed in to the service, rather than playing locally
```

**Invariants**
- An offline account has an empty profile identifier and a locally chosen name; an online
  one has both from the service.
- The name of an online account may not be changed by the game. Enforced, not assumed.
- The clan fields are always empty and the leader flag always false. See below.

## `load_account`

**Contract** — fills the account from the locally signed-in profile, if there is one.
Reaches through the main menu for the login manager and the profile store. A player who is
not signed in gets an empty name, no profile number and the offline flag — reported as a
warning, not an error, because playing without an account is supported.

```text
FUNCTION load_account()
  profile = login_manager.current_profile
  IF profile EXISTS
    player_name    = profile.unique_nick
    online_account = profile.online
    profile_id     = profile.profile_id
  ELSE
    player_name = "" ; online_account = false ; profile_id = 0
  clan_name = "" ; clan_leader = false
```

**Notes** — the clan fields are unconditionally cleared here and are never set anywhere in
the tree, yet they are carried on the wire and displayed by the score screen. The clan
system was part of the matchmaking service and its client side was removed while its wire
presence stayed. **Unrecovered**: what populated them.

Reaching the account service through the *main menu* is the dependency to note. The menu
owns the login session, so an account cannot be loaded in a process with no menu — which is
exactly the dedicated-server case. A dedicated server therefore never loads an account of
its own and only ever imports other players'.

## `net_Import` / `net_Export`

**Contract** — the frozen wire form. Read and write in this exact order:

```text
RECORD AccountMessage            # the frozen layout
  profile_id   : int (32-bit)
  player_name  : text, zero-terminated
  clan_name    : text, zero-terminated
  clan_leader  : int (8-bit, 0 or 1)
  online       : int (8-bit, 0 or 1)
  award_count  : int (16-bit)
  awards       : award_count repetitions of two 16-bit values
```

**Invariants** — the award list is **read and discarded** on import and written as **empty**
on export. The two values per award are consumed to keep the reader's position correct and
nothing is kept. That is the whole of the achievement system's surviving footprint: the
count field must still be present or every message after this one in the same packet
misparses.

**Notes** — this is the clearest example in the game of a frozen format outliving its
feature. A rebuild that drops the award count from the layout cannot talk to the original
executable, which is the compatibility bar multiplayer is held to.

The two 16-bit values per award are an identifier and presumably a count or a tier. Nothing
in the tree reads them, so **the second field's meaning is unrecovered.**

## `skip_Import`

**Contract** — consumes exactly one account from a message and keeps nothing, advancing the
reader past it. Exists because a client receives the roster of players it has no slot for —
during a join, or for spectators it is not tracking — and must still parse past their
accounts to reach the rest of the packet.

**Invariants** — it must consume **byte for byte** what `net_Import` consumes, including the
award loop. The two routines are written separately and there is nothing that keeps them in
step; the strings are read into a fixed buffer here and into interned storage there, which
is the only difference. A rebuild should derive both from one layout description rather than
maintaining two walks.

## `set_player_name`

**Contract** — changes the nickname. **Fails hard when the account is online.** An online
nickname is the service's property and every other client's copy of it came from the
service, so a local rename would silently desynchronize the roster. The refusal is a hard
assertion rather than a returned error because there is no caller that should be trying.
