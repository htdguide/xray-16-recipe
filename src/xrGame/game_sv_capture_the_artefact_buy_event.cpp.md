# src/xrGame/game_sv_capture_the_artefact_buy_event.cpp

> The asynchronous buy cycle: a dead player shops while he waits for his wave, his respawn is held until he closes the menu, and what he bought survives the respawn that would otherwise overwrite it.

**Needs** — [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md)
**Tier floor** — T3: message decoding and a three-state flag per client

## Purpose

Split out of [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) for
compilation reasons rather than design ones, and a rebuild folds it back. It is worth a page
anyway because it holds the whole of one decision the other modes did not make: **buying is
asynchronous with respawning.**

Deathmatch's dead player shops and is then respawned by whatever brings him back, and if the
two collide the purchase is lost. Capture the artefact instead tracks, per client, whether the
buy screen is open, and defers the respawn until it closes. Three small routines and two flags
implement it, and the reason they are worth reading is that the naive version — respawn on the
wave, apply the purchase when it arrives — produces a player who spawns with the previous
life's kit and then has it replaced under him a second later.

## State

```text
RECORD BuyCycleState                     # fields of the session; declared in the header twin
  buy_menu_states : map<client, BuyMenuState>
  dead_buyers     : map<client, int>     # 1 once this dead player has bought since dying
  not_free_ammo   : text                 # a list of sections excluded from the free magazine

ENUM BuyMenuState
  CLOSED            # the client reported the menu closed
  OPEN              # the client reported it opened; only a dead player may
  READY_TO_SPAWN    # a wave arrived while it was open; spawn him when he closes it
```

Invariants:

- **The two flags answer two different questions about the same screen.** The state says
  *where in the open-shop-close cycle this client is*; the buyer flag says *has he committed
  a purchase since he died*. The first drives when to respawn him, the second drives whether
  the respawn may overwrite his item list with the rank's defaults. Neither can be derived
  from the other.
- **The buyer flag is cleared on death, not on respawn.** It must be, because the respawn is
  the thing that reads it.
- Both tables are keyed by the client record rather than by the player, so an entry survives
  a player record being cleared and is discarded wholesale at round start.

## `OnPlayerBuyFinished`

**Contract** — applies a client's completed purchase. Destroys everything the player currently
carries, clears his item list, reads the money delta and the packed item list out of the
message, and records both. A **living** player's weapons are spawned at once; a **dead**
player's are not — instead he is marked as having bought, and they are spawned when he
respawns. Ends by telling the client its buy menu may be reopened.

```text
FUNCTION on_player_buy_finished(client, message)
  player = the player record for client
  actor  = the player's server object          # absent while permanently dead
  destroy every item the player carries
  clear the player's item list

  player.last_buy_amount = message.read_int32()     # the money delta, SIGNED
  count = message.read_int16()
  FOR EACH of count items
    group = message.read_byte()
    index = message.read_byte()
    player.item_list.append((group shifted up 8 bits) combined with index)

  IF the player is permanently dead
    dead_buyers[client] = 1                          # see Invariants
  ELSE
    spawn_weapons_for_actor(actor, player)
  tell the client its buy menu may be reopened
```

**Invariants** — the item encoding is **a packed pair of bytes in one sixteen-bit value**: the
low byte indexes the mode's price table, the high byte carries the weapon's addon flags (scope,
launcher, silencer). That packing is the loadout representation everywhere in the session — in
the player record, in the default-items list, in the spawn path — so a rebuild must either keep
it or change all of them together.

**The money delta is applied when the weapons are spawned, not here.** That is what lets a dead
player revise his purchase as many times as he likes without being charged each time: only the
last list he committed before respawning is ever paid for.

**The purchase is destructive before it is constructive.** Everything the player has is
destroyed first and the world is rebuilt from what the client sent. So a message that is
truncated or malformed leaves the player with nothing.

**Notes** — **the message is trusted.** The server destroys what the player had and rebuilds from
what the client sent, *including the price the client claims to have paid*. Everything
preventing a fabricated loadout lives on the client. A rebuild should price the list server
side from the same table the menu was built from — the table is already loaded here, so the
check costs a lookup per item.

The same hole exists in deathmatch's version of this routine, so it is a property of the buy
protocol rather than of this mode.

The buy-menu-may-reopen notification is sent by **reusing the menu-closed event** as its
carrier. That is not a harmless naming slip: the client's handler for that event is what marks
the menu usable, so the two meanings are genuinely the same message. A rebuild should give it
its own name.

## `OnPlayerOpenBuyMenu` / `OnPlayerCloseBuyMenu` / `OnCloseBuyMenuFromAll`

**Contract** — the state machine. Opening is accepted only from a permanently dead player and
records the open state. Closing is accepted only from a client that has an entry, is not a
spectator, and is still permanently dead; if that entry says a wave arrived while the menu was
open, the player is respawned now. The bulk clear discards every entry and is run at round
start.

```text
FUNCTION on_player_open_buy_menu(client)
  IF the player is not permanently dead: RETURN      # a living player buys synchronously
  buy_menu_states[client] = OPEN

FUNCTION on_player_close_buy_menu(client)
  entry = buy_menu_states[client] ; IF none: RETURN
  IF the player is a spectator: RETURN               # a spectator has nothing to buy for
  IF the player is not permanently dead: RETURN      # he has already been respawned
  IF entry is READY_TO_SPAWN
    respawn_client(client)                           # the deferred wave, delivered now
  entry = CLOSED

FUNCTION set_ready_to_spawn_player(client)           # called by the wave
  buy_menu_states[client] = READY_TO_SPAWN
```

**Invariants** — **a player shopping when his wave arrives loses no time.** The wave marks him
and he spawns the instant he closes the menu. Without the intermediate state he would either
be respawned with the screen open — losing the purchase to the respawn's default loadout — or
skipped entirely until the next wave, which at the default interval is a punishment for
shopping.

Closing is a **transition, not a toggle**: the state moves to closed whether or not a respawn
happened, so a second close message is inert.

**Notes** — the liveness and spectator tests appear twice each, once as an assertion that stops
a debug build and once as a plain return. The pair says the situation is believed impossible
*and* is handled; the source's own comments name both as things a client must never send. A
rebuild should keep the handling and drop the assertion — these are messages from an untrusted
end, so "impossible" is not a property the server can rely on.

An entry is created only by the open path, so a client that never opens the menu is never in
the table and the wave respawns it directly. That is the common case and it costs nothing.

A client that disconnects with the menu open leaves a stale entry keyed by a record that no
longer exists. Nothing looks it up again, and the round start clears them all.

## `CheckIfPlayerInBuyMenu`

**Contract** — true when the client has an entry in either the open or the ready-to-spawn state.
This is the test the wave consults before respawning anyone.

**Invariants** — **ready-to-spawn counts as "in the menu"**, which is what stops a second wave
respawning a player the first wave already deferred. Without it, a player who shops across two
wave intervals is respawned by the second one with his screen still open.

## `CanChargeFreeAmmo`

**Contract** — true unless the named ammunition section appears in this mode's not-free list,
read from configuration at creation. The answer governs whether a player who bought a weapon
but no rounds is given a full magazine anyway.

**Notes** — the membership test is a **substring search over the whole list as one string**, not
a comparison against tokenised names. A section whose name is a prefix of another's is
therefore wrongly treated as not free. A rebuild should split the list on its separators and
compare whole names; the intent is plainly a set. Deathmatch has the identical routine with
the identical flaw, which is how the pattern got copied here.

The free magazine is what stops the buy menu producing an unusable purchase — a weapon with no
ammunition is worse than no weapon, because it occupies the slot. The exclusion list exists so
that the expensive ammunition types cannot be obtained this way.

The routine asserts that the list was loaded and then tolerates its being empty, which is the
right shape: an empty list means everything is free, and that is a legitimate configuration.
