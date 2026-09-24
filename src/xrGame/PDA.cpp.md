# src/xrGame/PDA.cpp

> The personal data assistant: an inventory item that senses which characters are nearby and tells its owner, which is what populates the contact list and the map.

**Needs** — [`PDA.h`](PDA.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`Actor.h`](Actor.h.md) · [`Entity.h`](Entity.h.md) · [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`specific_character.h`](../xrServerEntities/specific_character.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`PDA.h`](PDA.h.md)
**Tier floor** — T3: a proximity query at the scheduler's degraded rate and two callbacks

## Purpose

The contact list, the map markers and the "somebody is nearby" notifications all come from one
place: a device the player carries that uses the engine's *touch* sense to notice other
characters within a radius. The important part is that it is a real item — it can be dropped,
taken, or belong to somebody else, and it only works for the person it was issued to.

A second, smaller job: a found device can carry a recorded message, delivered as a script call
when the player plays it back. That is how story audio logs are implemented.

## State

```text
RECORD Pda                                 # on top of the inventory item
  radius              : real               # sensing radius, from configuration
  original_owner_id   : entity id          # who this device was issued to
  specific_character  : text               # that owner's character record
  full_name           : text               # "<item name> <owner name>", built once
  turned_off          : bool
  active_contacts     : list<object>       # the sensed set, filtered to living humans
  functor_name        : text               # the script function played back, or empty
```

**Invariants**

- The device is on **only while held by its original owner**. Picking up somebody else's device
  gives you the object and its recorded message, not their contact list. This is the class's
  central rule and the reason the original owner's identity is part of the spawn record rather
  than assigned on pickup.
- The sensed set and the active-contact list are separate: the first is what the touch sense
  reports, the second is that set filtered to living non-monsters. Nothing outside reads the
  raw set.
- `full_name` is built once, on first attachment to the original owner, and then *saved*. A
  device recovered from a corpse must still name its dead owner, who may no longer exist.

## `net_Spawn`

**Contract** — spawns as an inventory item and takes the original owner's identifier and
character record from the spawn record. Fails hard if the spawn record is not a device record.

**Notes** — the return ignores the base class's result and always reports success. A base spawn
failure is therefore swallowed, which is a defect.

## `Load`

**Contract** — reads the sensing radius, and optionally the name of the script function this
device plays back. An absent playback function means the device has no recorded message.

## `shedule_Update`

**Contract** — the low-rate update. Keeps the device at its owner's position, and — only when
it is on and its owner is the entity the player is currently controlling — runs the proximity
sense and rebuilds the contact list. Turns itself off if its owner has died.

```text
FUNCTION shedule_Update(dt)
  inherited shedule_Update(dt)
  IF no parent THEN RETURN
  position = parent.position

  IF turned on AND the parent is the entity the player controls THEN
    IF the parent is not a living entity THEN TurnOff(); RETURN
    run the touch sense at (position, radius)
    UpdateActiveContacts()
  END IF
```

**Invariants** — the sense runs *only for the player's own device*. Every other character in
the world carries one and none of them sense anything, because nothing reads their contact
lists. A rebuild that runs the sense for every device will pay a proximity query per character
per update for nothing.

**Notes** — running at the scheduler's degraded rate is deliberate and visible: contacts appear
a moment after a character comes into range, and the delay grows with distance and load. That
is the engine's general update-budget mechanism applied here rather than a decision of this
class.

## `feel_touch_contact`

**Contract** — the filter deciding which objects the touch sense should track. Accepts every
monster, and every *other* character that carries a device. Rejects the owner's own device and
everything else.

```text
FUNCTION feel_touch_contact(object) -> bool
  IF object is a living entity AND is a monster THEN RETURN true
  IF object is an inventory owner THEN
    IF this device is that owner's own device THEN RETURN false     # do not sense yourself
    RETURN whether the object is a living entity
  END IF
  RETURN false
```

**Notes** — monsters are admitted here and then discarded by the contact filter downstream.
Tracking them and then dropping them looks wasteful, and it is; the sensed set is used by
nothing else. A rebuild should reject them here.

## `feel_touch_new` · `feel_touch_delete`

**Contract** — a character entering or leaving sensing range. Each forwards the event to the
device's *owner*, who maintains the actual contact record — first-meeting flags, reputation,
the map marker. The delete path tolerates the device having no owner, because a device dropped
while contacts are in range will lose them after it is detached.

**Invariants** — the owner, not the device, holds the contact history. The device is only the
sensor. That separation is what lets contact history survive losing the device.

## `UpdateActiveContacts`

**Contract** — rebuilds the contact list from the sensed set, keeping living non-monsters.
Clears and refills; the list is never incrementally maintained.

**Notes** — it dereferences each sensed object as a living entity without checking, relying on
the sense filter having admitted only living entities and monsters. That holds today and is
fragile.

## `OnH_A_Chield`

**Contract** — the device gains an owner. If that owner is the original owner, the device turns
on and — once, on first attachment — composes its display name as the item's name followed by
the owner's name. Asserts that the device was off.

**Notes** — the composed name is what makes a recovered device read as "PDA — Strelok" in the
inventory rather than as a generic item. Composing it at attachment time rather than on demand
is what allows the name to be saved and to outlive the owner. The commented-out alternative in
the original composed it lazily from the *character record* instead, which would have survived
the owner's destruction without needing to be saved; it was abandoned.

## `OnH_B_Independent`

**Contract** — the device loses its owner. Turns off unconditionally, whoever the owner was.

## `net_Destroy`

**Contract** — turns off, clears the sensed set and the contact list, then destroys as an item.

**Invariants** — clearing the sensed set *without* running the leave callbacks means the
owner's contact records are not updated. That is correct here: the device is being destroyed
along with everything else, and firing callbacks into objects that may already be gone is
worse.

## `save` · `load`

**Contract** — persist and restore the composed display name on top of the base item's state.
Nothing else is saved: the original owner comes from the spawn record, the contact list is
rebuilt by sensing, and the on/off state follows from who is holding it.

## `PlayScriptFunction` · `CanPlayScriptFunction`

**Contract** — call the configured script function, or report whether there is one. The call
fails hard if the named function does not exist, so a device configured with a missing
function crashes on playback rather than doing nothing.

**Notes** — this is the entire mechanism behind found audio logs and story messages: the device
names a script function and playing the device calls it. Everything the message *does* —
showing text, playing audio, granting knowledge — is script.

## `GetOriginalOwner` · `GetOwnerObject` · `GetPdaFromOwner` · `ActivePDAContacts`

**Contract** — resolve the original owner's identifier to a live object (returning nothing if
that character no longer exists), fetch the device belonging to a given character, and map the
contact list from characters to their devices.

**Invariants** — a contact whose owner carries no device is silently dropped from the device
list. That is why the contact count and the device-contact count can differ.
