# src/xrGame/InventoryOwner.h

> Declares the character mixin — inventory, personality, money, trade, conversation and knowledge — implemented across [`InventoryOwner.cpp`](InventoryOwner.cpp.md) and its siblings.

**Needs** — [`xrServerEntities/InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md) · [`pda_space.h`](pda_space.h.md) · [`attachment_owner.h`](attachment_owner.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`ActorBackpack.h`](ActorBackpack.h.md) · [`inventory_owner_inline.h`](inventory_owner_inline.h.md)
**Used by** — [`AI_PhraseDialogManager.cpp`](AI_PhraseDialogManager.cpp.md) · [`Actor.cpp`](Actor.cpp.md) · [`Actor.h`](Actor.h.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Artefact.cpp`](Artefact.cpp.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`HUDTarget.cpp`](HUDTarget.cpp.md) · [`InfoDocument.cpp`](InfoDocument.cpp.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`InventoryOwner.cpp`](InventoryOwner.cpp.md) · [`PhraseScript.cpp`](PhraseScript.cpp.md) · [`UIGameCustom.cpp`](UIGameCustom.cpp.md) · [`UIGameDM.cpp`](UIGameDM.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · _and 19 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CInventoryOwner`, the half of an entity that makes it a *character* rather than a
body: it can carry, trade, talk, know things and have a standing in a faction. Everything
that can be looted or spoken to in this game is one.

Substance is split across two files, and the split is worth knowing before reading either:
the lifecycle, trade, standing and item hooks are in
[`InventoryOwner.cpp`](InventoryOwner.cpp.md); the knowledge surface — receiving, losing and
testing for info portions — together with the network events, is in
[`inventory_owner_info.cpp`](inventory_owner_info.cpp.md).

The shape this header fixes: it derives from the attachment owner (a character wears things
on its body) and is mixed into an entity *beside* the movement and AI halves, never above
them. `cast_inventory_owner` is the capability query by which the rest of the game discovers
that some object is a character at all — the recurring alternative to asking its type.

Exported units:

- `CInventoryOwner` — the mixin.
- `cast_inventory_owner` — the capability query.
- `_construct` / `net_Spawn` / `net_Destroy` / `Init` / `Load` / `reinit` / `reload` /
  `OnEvent` / `save` / `load` / `UpdateInventoryOwner` — the lifecycle, in the order an
  entity runs it.
- `m_inventory` / `inventory()` — the carried belongings, public and reached into directly
  by screens and scripts.
- `CanPutInSlot` — the owner's veto over a slot placement; the base permits everything.
- `GetPDA` / `GetOutfit` / `GetBackpack` — the three fixed-slot lookups.
- `MaxCarryWeight` / `GetWeaponAccuracy` — the two tunables a specific owner recomputes.
- `OfferTalk` / `StartTalk` / `StopTalk` / `IsTalking` / `GetTalkPartner` /
  `EnableTalk` / `DisableTalk` / `IsTalkEnabled` / `bDisableBreakDialog` — the conversation
  handshake and its permission.
- `StartTrading` / `StopTrading` / `IsTrading` / `GetTrade` / `EnableTrade` /
  `DisableTrade` / `IsTradeEnabled` / `EnableInvUpgrade` / `DisableInvUpgrade` /
  `IsInvUpgradeEnabled` — the trade session and its two permissions.
- `trade_parameters` / `trade_section` / `AllowItemToTrade` / `deficit_factor` /
  `buy_supplies` / `sell_useless_items` / `on_before_sell` / `on_before_buy` — what this
  owner deals in, how scarce it is, and how its stock is replenished.
- `OnReceiveInfo` / `OnDisableInfo` / `TransferInfo` / `HasInfo` / `GetInfo` /
  `m_known_info_registry` / `DumpInfo` — the knowledge surface: gaining a fact, losing one,
  granting one through the server, and testing for one.
- `Name` / `IconName` / `object_id` / `is_alive` — identity.
- `get_money` / `set_money` / `InfinitiveMoney` — money, with an unbankruptable mode.
- `CharacterInfo` / `SpecificCharacter` / `Community` / `Rank` / `Reputation` / `Sympathy` /
  `SetCommunity` / `SetMonsterCommunity` / `SetRank` / `ChangeRank` / `SetReputation` /
  `ChangeReputation` / `SetIcon` — the personality record and standing, each setter also
  writing through to the server record.
- `NewPdaContact` / `LostPdaContact` — communicator contact notifications.
- `renderable_Render` — draws the held item under the body attachments.
- `OnItemTake` / `OnItemBelt` / `OnItemRuck` / `OnItemSlot` / `OnItemDrop` /
  `OnItemDropUpdate` — the placement hooks the inventory calls back into.
- `spawn_supplies` / `use_bolts` / `item_to_spawn` / `ammo_in_box_to_spawn` — standard issue
  at spawn.
- `unlimited_ammo` — demanded of every implementor with no base answer; whether this owner's
  weapons consume ammunition.
- `on_weapon_shot_start` / `on_weapon_shot_update` / `on_weapon_shot_stop` /
  `on_weapon_shot_remove` / `on_weapon_hide` — recoil and aim feedback hooks.
- `use_default_throw_force` / `missile_throw_force` / `use_throw_randomness` — throwing.
- `can_use_dynamic_lights` / `NeedOsoznanieMode` / `CanPlayShHdRldSounds` /
  `SetPlayShHdRldSounds` — presentation switches, chiefly separating the player (who hears
  and lights everything) from everyone else.
- `deadbody_can_take` / `deadbody_closed` and their status readers — whether this owner's
  corpse may be looted or opened. Settable only once dead.
- `OnFollowerCmd` — the squad-order hook; meaningful only for a stalker.

## Notes

`Init` is declared here and has no definition anywhere in the tree. Nothing calls it.

`unlimited_ammo` is the only thing this mixin *demands* of an implementor. Everything else
has a base answer. That single demand is the clearest statement of what the mixin cannot
decide for itself: whether this character is bound by ammunition economy at all.
