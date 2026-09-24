# src/xrGame/InventoryOwner.cpp

> The character half of an entity: an inventory, a personality record, money, trade and conversation state, and the knowledge it has received.

**Needs** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`Inventory.h`](Inventory.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`trade.h`](trade.h.md) · [`trade_parameters.h`](trade_parameters.h.md) · [`purchase_list.h`](purchase_list.h.md) · [`PDA.h`](PDA.h.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`Bolt.h`](Bolt.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`AI_PhraseDialogManager.h`](AI_PhraseDialogManager.h.md) · [`Level.h`](Level.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: composition, lifecycle ordering and message framing

## Purpose

Anything in this game that can be talked to, traded with, or looted is an **inventory
owner**: the player, every stalker, every trader, and every corpse of one. This file is the
shared half of all of them — the part that is not about how they move or decide.

It owns five things that together make a character: an inventory, a personality record
(name, faction, rank, reputation, portrait, opening dialog), money, a trade session, and a
conversation partner. It is mixed into an entity alongside the movement and AI halves rather
than derived from them, because the player and a stalker share all of this and share nothing
of how they are driven.

The same-shaped state lives on both sides of the server/client divide. Rank, reputation,
faction and the dead-body flags are *written through* to the server record on every change,
because those are what a save file and the alife simulation carry; the inventory itself is
not, because items are separate entities with their own records.

## State

```text
RECORD InventoryOwner
  inventory             : Inventory
  character_info        : CharacterInfo    # faction, rank, reputation, portrait, start dialog
  game_name             : text             # the display name; NOT the section name
  money                 : int
  trade                 : Trade            # the trade session object, one per owner
  trade_parameters      : TradeParameters  # what this owner will buy and sell, from config
  purchase_list         : optional<PurchaseList>   # restock model, created on first use
  trading               : bool
  talking               : bool
  talk_partner          : optional<InventoryOwner>
  allow_talk            : bool
  allow_trade           : bool
  allow_inv_upgrade     : bool
  disable_break_dialog  : bool
  known_info_registry   : InfoPortionRegistry   # the facts this character has received
  pending_active_slot   : int   # restored from a save, applied when the item arrives
  play_show_hide_reload_sounds : bool
  need_osoznanie_mode   : bool
  deadbody_can_take     : bool  # may this corpse be looted
  deadbody_closed       : bool  # may this corpse be opened at all
  item_to_spawn         : text  # a one-shot item this owner should be given
  ammo_in_box_to_spawn  : int
```

**Invariants**
- The dead-body flags are only settable while the owner is *not* alive. Both routines return
  silently on a living owner, so the "can this corpse be looted" state cannot be pre-set.
- `pending_active_slot` is the one piece of load state that cannot be applied at load time —
  see `load` and `OnItemTake` below.
- The personality record must be initialized from the server record before anything asks
  this owner its name, faction or rank; the spawn refuses outright if the record is not a
  character record.

## Construction and destruction

**Contract** — the inventory, the knowledge registry and the personality record are created
with the owner and live exactly as long as it. The trade session and trade parameters are
not: they are created at spawn, because the trade parameters depend on the configuration
section, which is not known until then. Talk and trade are both enabled by default; a
character that should not be approachable must be disabled explicitly.

## `Load`

**Contract** — pulls the two owner-level tunables from the entity's configuration section: a
carry-weight override that replaces the global default, and one behavioural flag. An absent
weight key leaves the global limit in place, which is how most characters are configured.

## `reload` / `reinit`

**Contract** — the two reset points of the entity lifecycle. `reload` empties the inventory,
re-points it at this owner, re-enables slots, and zeroes money, trade and conversation.
`reinit` clears only the pending-supply fields.

**Invariants** — the inventory's back-pointer to its owner is established here and nowhere
else. It is a reset step rather than a construction step because the inventory outlives
individual reloads of the same entity.

## `net_Spawn`

**Contract** — brings the character half online from the server record. Must run *after* the
game object's own spawn. The single-player and multiplayer paths diverge completely: single
player demands a real character record and synchronizes from it; multiplayer synthesizes a
generic player personality from a fixed configuration section and takes only the display
name from the record.

```text
FUNCTION net_spawn(server_record) -> bool
  IF no trade session yet
    create one bound to this owner
  replace trade parameters with ones built from trade_section()

  IF NOT single_player
    character_info.load("mp_actor"); character_info.init("mp_actor")
    game_name = server_record.replacement name, else the object's own name
    RETURN true

  record = server_record as a character record
  IF record IS none
    RETURN false                       # refuses to spawn: not a character
  REQUIRE record.character_profile is non-empty

  character_info.init(record)          # faction, rank, reputation, portrait, start dialog
  known_info_registry.bind(record.id)  # attach this owner's knowledge to the alife registry

  IF this owner runs conversations AND has no opening dialog yet
    take the opening dialog from the personality record, as both the
    current and the default — the default is what a script resets to
  game_name = record.character_name
  deadbody_can_take = record.deadbody_can_take
  deadbody_closed   = record.deadbody_closed
  RETURN true
```

**Notes** — the opening dialog is set only if empty, so a script that assigned one before
the spawn keeps it. Storing it twice, as current and default, is what makes "give this
character a temporary dialog and then put it back" expressible without the script having to
remember the original.

Rebuilding the trade parameters unconditionally — rather than keeping existing ones — matters
because the same entity can respawn under a different section after a level change.

## `net_Destroy`

**Contract** — clears the inventory and empties the hands. Note the asymmetry with the
spawn: the trade session and trade parameters are *not* released here, only at destruction,
so an owner that goes offline and comes back reuses them.

## `save` / `load`

**Contract** — the character half's saved state, in this order: the active slot as a single
byte, the personality record, the display name, the money. The order is frozen by the save
format.

**Invariants** — the active slot is saved as one byte, which caps the slot count the save
format can express at 255 even though slots are counted more widely everywhere else.

The load cannot restore the active slot directly. At load time the inventory is empty — the
items are separate entities that arrive afterwards — so the slot number is *remembered* and
applied later, by `OnItemTake`, at the moment the item that belongs in it arrives. Only the
"nothing in hand" case can be applied immediately.

## `UpdateInventoryOwner`

**Contract** — the character half's per-frame step, called from the entity's own update.
Runs the inventory's update and the trade session's, then enforces two teardown conditions:
a dead owner stops trading, and a conversation ends when either the partner has stopped
talking or this owner has died.

**Notes** — conversation teardown is polled from both sides rather than pushed from one.
That is what makes it robust to either party dying, being destroyed, or leaving the level:
whichever side is still updating notices the other has gone.

## `OfferTalk` / `StartTalk` / `StopTalk` / `IsTalking`

**Contract** — the conversation handshake. An offer is accepted when this owner permits talk
at all and both parties are alive; it then starts the conversation on this side. Stopping
clears the partner and, if the conversation screen is currently showing, closes it.

**Notes** — the relationship check is written and commented out: an enemy can be talked to.
Whether that is a bug or a deliberate relaxation is not recoverable from the source; the
commented line would have refused a conversation with anyone hostile.

Starting a conversation takes a "start trade as well" parameter and ignores it.

## `StartTrading` / `StopTrading` / `IsTrading` / `GetTrade`

**Contract** — the trade flag and the session object. Stopping trade hides the inventory
screen as a side effect. Asking for the session before the spawn created it is a hard
failure rather than an empty result.

## `renderable_Render`

**Contract** — draws the item in the hands *before* the owner's attached objects. The order
is load-bearing for the first-person view, where the held weapon must be composited under
the attachments rather than over them.

## `OnItemTake` / `OnItemDrop` / `OnItemBelt` / `OnItemRuck` / `OnItemSlot` / `OnItemDropUpdate`

**Contract** — the five placement hooks the inventory calls, each firing a script callback
and each deciding whether the item is *attached* to this owner. Attachment is what makes an
item visible on the character's body and included in its bounding volume.

```text
on_item_take : callback(item taken); attach
on_item_slot : callback(item to slot); attach
on_item_ruck : callback(item to ruck); DETACH   # a rucked item is not on the body
on_item_belt : callback(item to belt); no change
on_item_drop : callback(item dropped); detach
```

**Notes** — the belt hook deliberately neither attaches nor detaches, because an item
reaches the belt only from a slot or the ruck, both of which already left the attachment in
the right state.

`OnItemTake` also carries the deferred active-slot restore from a loaded save: when the
arriving item lands in the slot the save recorded as active, the hands are filled and the
pending slot is cleared. That is why a loaded game shows the right weapon in hand despite
the inventory being empty at load time.

## `MaxCarryWeight`

**Contract** — the inventory's configured limit plus the bonus from the worn outfit, if any.
Outfits are the only source of extra capacity.

## `GetPDA` / `GetOutfit` / `GetBackpack`

**Contract** — the three fixed-slot lookups. Each is "whatever is in this named slot, if it
is of the right kind". They are methods rather than fields because the item can be removed,
destroyed or replaced at any time and a cached handle would go stale.

## `spawn_supplies`

**Contract** — gives a newly spawned character its standard-issue belongings. Monsters are
exempt entirely. Everyone else who uses bolts gets one — bolts are the throwable used to
probe for anomalies and are effectively infinite. Additionally, **when the alife simulation
is not running**, each character is given a personal communicator stamped with its own
identity, spawned directly through the network rather than through the ordinary path.

**Notes** — the communicator is spawned only without alife because with alife running the
simulation creates it as part of the character's offline inventory. The direct network spawn
followed by an immediate release of the server record is how a server-side object is created
outside the alife path; it is a workaround for having two spawn mechanisms, and a rebuild
with one needs neither branch.

## `SetCommunity` / `SetMonsterCommunity` / `SetRank` / `ChangeRank` / `SetReputation` / `ChangeReputation`

**Contract** — the four standing values, each written to the personality record *and*
through to the server record so it survives a save. Every one of them silently does nothing
when no server record exists for this entity, which is the case for a purely client-side
object.

**Invariants** — changing faction additionally re-teams a living entity, because team
membership is what the combat code consults for friend-or-foe, and it must follow the
faction immediately rather than at the next reload.

**Notes** — the delta forms are defined in terms of the absolute forms, so a delta applied
to an owner with no server record is silently lost rather than accumulating.

Faction, rank and reputation each write through to the record only if it is specifically a
*trader* record. That coupling of "has a personality" to "is a trader record" is historical:
the server-side type that carries character data is named for the trader.

## `set_money`

**Contract** — sets the money and optionally broadcasts it. A character flagged as having
infinite money never loses any: the assignment becomes a maximum, so selling to such a
character raises their stock but buying never drains it. That is how the shipped traders are
made unbankruptable without special-casing every transaction.

## `buy_supplies` / `sell_useless_items` / `deficit_factor` / `trade_section` / `AllowItemToTrade`

**Contract** — the restock model. A trader's stock is defined in configuration and processed
through a purchase list, which also answers how scarce a section currently is — the scarcity
factor feeds the price. The trade section defaults to a shared one and is overridden per
entity type.

`sell_useless_items` is the counterpart: it **destroys** everything the character should not
be seen carrying, sparing bolts, weapons that are in a slot, and the character's own
personal communicator. It is destruction rather than a sale despite the name — the items
vanish, no money changes hands.

**Notes** — sparing "weapons in a slot" rather than "weapons" means a spare rifle in the
ruck is destroyed while the equipped one survives. The communicator exemption is by
*original owner*, so a communicator looted from someone else is destroyed — which is the
mechanic that stops recovered communicators piling up.

## `deadbody_can_take` / `deadbody_closed`

**Contract** — whether a corpse may be looted and whether it may be opened at all. Both
refuse on a living owner, and both broadcast the *pair* of flags together, because the
receiving side reads them as one state.

## `Name` / `IconName` / `object_id` / `is_alive`

**Contract** — the identity surface. The display name comes from the stored game name rather
than the personality record — a deliberate divergence, since a script may rename a character
without rewriting its personality. The portrait comes from the personality record. Liveness
is asked of the entity half and is a hard failure on an owner that is not a living entity,
which means a non-living inventory owner cannot exist.

## `on_weapon_shot_start` / `on_weapon_shot_update` / `on_weapon_shot_stop` / `on_weapon_shot_remove` / `on_weapon_hide` / `NewPdaContact` / `LostPdaContact` / `GetWeaponAccuracy` / `use_default_throw_force` / `use_throw_randomness`

**Contract** — the hooks a specific kind of owner overrides. The base does nothing, reports
zero spread, and opts into default throwing behaviour. The base spread of zero is not a
neutral default: an owner that does not override it is perfectly accurate.

## `missile_throw_force`

**Contract** — deliberately has no base answer. Reaching it is a defect, because a throw can
only be issued by an owner that has overridden it.
