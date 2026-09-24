# src/xrGame/ui/UIMpTradeWnd_wpn.cpp

> Weapon attachments: the seven quick buttons, and the attach/detach layer that asks the weapon
> itself what it will accept.

**Needs** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`WeaponMagazinedWGrenade.h`](../WeaponMagazinedWGrenade.h.md) · [`Restrictions.h`](Restrictions.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md)
**Used by** — [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md)
**Tier floor** — T3.

## Purpose

Attachments are not items in a list — they are *bits on the weapon*. This file is the whole of
that translation: a request to add a scope becomes buying a scope record, setting a bit, and
destroying the record; a request to remove one becomes clearing the bit, re-creating the record,
refunding it and destroying it again.

## The quick buttons

**Contract** — seven handlers, one per attachment or ammunition kind, each operating on whatever
is in the pistol or rifle slot and doing nothing when that slot is empty.

- **The three attachment buttons** (silencer on pistol, silencer, scope and launcher on rifle)
  **toggle**: attached means detach and refund, unattached and attachable means buy and attach.
  A kind the store does not sell is silently declined.
- **The three ammunition buttons** buy one round of the weapon's first ammunition type, or its
  *second* while the modifier key is held. The launcher-ammunition button buys the first
  launcher round and requires a launcher-capable weapon.
- Removing a launcher additionally clears the rifle's ammunition list and re-derives it, because
  the launcher's ammunition types have just stopped applying.

**Notes** — the modifier key selecting the second ammunition type is undiscoverable and
unbound; it exists because a weapon may take two rounds and there is one button. A rebuild
should offer both explicitly.

## `TryToAttachItemAsAddon`

**Contract** — decide whether a just-bought item is an attachment and, if so, where it goes.
With a named parent, attach to that parent if it will take this kind. With no parent, try the
rifle first and then the pistol, attaching to the first that will accept it. Report whether
anything was attached.

**Notes** — the **auto-attach preference is rifle before pistol**, which is what makes buying a
silencer while holding both put it on the rifle. The loop that tries the two returns outright
when the first slot is empty instead of continuing to the second, so a player holding only a
pistol gets no auto-attach at all. That is a real behavioural bug preserved here; the intent is
obvious and a rebuild should simply skip empty slots.

## Attach, detach, and the attachment mask

**Contract** —

```text
FUNCTION is_attached(record, kind)   # the weapon both accepts the kind and has one on
FUNCTION can_attach(record, kind)    # accepts the kind and does NOT have one on
FUNCTION attach(record, kind)        # set the kind's bit in the weapon's attachment mask
FUNCTION detach(record, kind)        # clear the bit, then CREATE a record for the
                                     # attachment in the "own" state and return it
FUNCTION addon_section(record, kind) # the weapon's own answer for which section that kind is
FUNCTION kind_of(section)            # by the section's restriction group: "scp", "sil", "gl"
```

**Notes**

- **A weapon names its own attachments.** The section a scope on *this* rifle would be is asked
  of the rifle, not looked up in a table, which is how one button serves every weapon.
- **What kind of attachment an item is comes from its restriction group**, matched against three
  fixed short names. So the classification that caps how many of a thing you may carry is the
  same classification that decides where it attaches. That is an economical reuse and a fragile
  one: renaming a group in the shipped data silently turns a scope into a non-attachment.
- Detaching *creates* a record rather than recovering one, because the record was destroyed when
  the attachment went on. The new record is in the *own* state, so the seller path treats it as
  something the player already had, which is exactly right for a refund.

## `SellItemAddons`

**Contract** — for one attachment kind on one weapon: if attached, detach it, refund its price
at the current rank, and destroy the resulting record. Removing a grenade launcher additionally
sells every bought round of every launcher ammunition type the weapon has. Items that are not
weapons are ignored.

**Notes** — the cascade is the rule that "ammunition follows its weapon" applied one level
down: launcher rounds are worthless without the launcher, so removing it refunds them too
rather than leaving them in the bag. Only *bought* rounds are swept; rounds the player arrived
with are left alone.

## The two free helpers

**Contract** — read and write a record's attachment mask, going straight to the weapon behind
the cell widget and answering zero for anything that is not a weapon. They exist so that the
loadout code can record and replay a mask without knowing about weapons at all.
