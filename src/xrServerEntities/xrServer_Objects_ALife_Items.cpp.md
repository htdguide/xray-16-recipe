# src/xrServerEntities/xrServer_Objects_ALife_Items.cpp

> Every carryable record: what a weapon remembers, what a dropped item sends over the wire, and the quantization change at version 122.

**Needs** — [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`clsid_game.h`](clsid_game.h.md) · [`xrMessages.h`](xrMessages.h.md) · [`alife_space.h`](alife_space.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: on-disk and on-wire layouts, including a quantized encoding.

## Purpose

The authoritative records for everything a creature can carry. Each states what the item
remembers across a save, what it puts on the wire when it is dropped or thrown, and — for
weapons — the ammunition, magazine, attachment and condition fields that the game layer
reads back.

These records are frozen: every shipped level's spawn file and every existing save game is
a stream of them. The chapter's central hazard appears here in the small, as a
quantization change at one format version that a reader must switch on.


## `CSE_ALifeInventoryItem`

The mixin every carryable record adds.

```text
RECORD InventoryItemMixin           # state payload
  condition : real                  # 0..1, from version 53
  upgrades  : list<text>            # installed upgrade section names, from version 124
```

Fields held but **not** serialized, read from the item's configuration section every time the
record is built: mass (`inv_weight`), cost (`cost`), the health and food values (`health_value`,
`food_value`, defaulting to zero), and the starting condition (`condition`, defaulting to 1).

**Invariants**

- **Only condition and upgrades survive a save.** Everything else about an item is its
  configuration section, so a patch that rebalances weapon weight changes every existing save
  game's inventory. That is deliberate and it is the reason an entity is (class, section).
- **The upgrade list is a set**: adding an upgrade that is already present is fatal, not
  ignored. A rebuild should decide whether to keep the hard failure; the invariant that the
  list has no duplicates is the part that matters.
- **The item's position is copied from the record into the physics state on every state
  read and write.** The record's placement is the authority; the physics state's position is
  a cache.

### The dropped-item update

```text
RECORD ItemUpdate
  header           : int (8-bit)    # low 5 bits a count, high 3 a mask; zero ends the record
  force, torque    : real x 3       # from version 122
  position         : real x 3
  orientation      : real x 4       # 32-bit reals from 122; four quantized bytes before
  angular_velocity : real x 3       # omitted when the mask says zero
  linear_velocity  : real x 3       # omitted when the mask says zero
  awake            : int (8-bit)    # from version 122; absent in a spawn-plus-update packet
```

**Version 122 is a change of encoding, not of presence.** Before it, the orientation
components were bytes quantized over 0..1, angular velocity bytes quantized over
0..20π, and linear velocity bytes quantized over −32..32 metres per second. From 122 all
three are full 32-bit reals, and force and torque appear for the first time. A reader must
branch on the version and *dequantize*; a writer only ever emits the modern form. The
quantization ranges are the frozen part: a rebuild reading an old save must use exactly
0..1, 0..20π and ±32, or the restored pose is wrong rather than merely imprecise.

The same masking rule as the physic object applies: the two velocity vectors are omitted when
zero and reconstructed as zero.

**`Net_Relevant`** — an item inside someone's inventory is never sent. A loose awake item
always is. A loose sleeping item is sent once on falling asleep, then on roughly one draw in
120 for one second, then never. (The physic object uses 40 draws and five seconds; items are
more numerous and settle faster.)

**`update_rate`** — one second, declared and currently unused: the throttle it was meant to
drive is commented out at the call site. **Unrecovered** — why it was disabled.

## `CSE_ALifeItem`

A dynamic visual object plus the item mixin; payload in that order.

**The binocular exception.** Between the record's own payload and the mixin's, a reader whose
class identifier is the binocular and whose version is below 37 consumes and discards two
16-bit values and a byte — the tail of the weapon record the binocular used to be. This is
the only place in the chapter where the *class identifier* participates in a version gate,
and it is the reason the class identifier must be known before the payload is parsed.

**`Net_Relevant`** — relevant when either parent says so, never when attached to a carrier,
and otherwise when the item is moving and its physics has not been explicitly frozen. Freezing
arrives as a game event (`GE_FREEZE_OBJECT`) and is cleared by the next update read.

## `CSE_ALifeItemWeapon`

```text
RECORD Weapon                       # state payload
  ammo_total     : int (16-bit)     # rounds carried for this weapon; default 90
  ammo_in_magazine: int (16-bit)
  weapon_state   : int (8-bit)
  addon_flags    : int (8-bit)      # from version 41
  ammo_type      : int (8-bit)      # index into the section's ammo class list, from 47
  grenades       : int (8-bit)      # packed, from version 123
```

```text
RECORD WeaponUpdate
  condition      : int (8-bit)      # quantized over 0..1 — the item's condition, re-sent
  weapon_flags   : int (8-bit)
  ammo_in_magazine: int (16-bit)
  addon_flags    : int (8-bit)
  ammo_type      : int (8-bit)
  weapon_state   : int (8-bit)
  zoomed         : int (8-bit)
```

```text
CONSTANT addon_scope            = 0x01
CONSTANT addon_grenade_launcher = 0x02
CONSTANT addon_silencer         = 0x04
```

**Invariants**

- **The grenade count byte packs two fields**: six bits of count, two bits of type. The
  comment states the consequence — at most 64 grenades, at most four grenade types — and both
  limits are real. Pack as `(type << 6) | count`; unpack with a mask of `0x3f`.
- **Which addons a weapon may carry is configuration, not record state.** Three keys
  (`scope_status`, `silencer_status`, `grenade_launcher_status`) each read a small integer
  meaning disabled, permanently attached, or attachable; the record's addon flags are only
  meaningful for the attachable ones, and the editor only offers those. A rebuild must read
  the three keys as integers, because shipped configuration writes them as numbers.
- **The condition is quantized to one byte in the update** and kept as a full real in the
  state. A weapon's condition therefore steps in 1/255 increments while being watched and is
  exact in a save.
- **Ammunition capacity and slot come from configuration** (`ammo_limit`, `slot`,
  `ammo_mag_size`), not from the record.

**`Net_Relevant`** — a weapon whose flag byte equals 1 is always sent. That value means "just
fired"; everything else falls through to the item rules.

**`OnEvent`** — a weapon-state-change event carries four bytes: the new state, a sub-state, a
new ammunition type and an elapsed count. Only the first is kept; the other three are read
and dropped. The event is a client-to-server notification whose remaining fields the server
does not trust. **Unrecovered** — whether the three dropped fields were ever consumed.

## Weapon variants

- **Magazined** — update gains one byte, the selected fire mode. State unchanged.
- **Magazined with grenade launcher** — update gains one byte, the launcher toggle, **written
  before the parent's payload**, not after. This is the only record in the chapter that
  prefixes rather than appends, and a rebuild that normalizes it breaks the protocol.
- **Shotgun** — update gains a count byte followed by that many ammunition-type bytes: the
  exact sequence of shells in the tube, which matters because a shotgun can be loaded with
  mixed rounds and they come out in order.
- **Auto shotgun** — identical to shotgun in every direction. A distinct class identifier and
  nothing else.

## `CSE_ALifeItemAmmo`

State and update each add one 16-bit field: rounds remaining. The box size comes from
configuration (`box_size`) and seeds the count at construction.

**Invariant** — an ammunition box with zero rounds may not go offline. It is about to be
destroyed and the simulation must not carry it away first.

## `CSE_ALifeItemTorch`

State adds nothing (and is skipped entirely below version 21). Update adds one byte of
flags: torch on, night vision on, attached to a carrier. An attached torch is always network
relevant, because its light must follow its owner.

## `CSE_ALifeItemPDA`

```text
RECORD PDA
  original_owner       : int (16-bit)   # from version 59; 0xFFFF = none
  specific_character   : text           # from version 98; two discarded integers at 90..97
  info_portion         : text           # likewise
```

Both strings were numeric indexes into an XML table before version 98 and are names after it.
The reader for 90..97 consumes eight bytes and yields empty strings — the old indexes cannot
be translated, so that data is simply lost. A rebuild does the same.

**The tools write empty strings** where the game writes the real ones, because the two text
fields depend on the character database that only the game links. The record's *shape* is
identical either way, which is what lets the same file compile into both.

## `CSE_ALifeItemDocument`

One info-portion name, a text string from version 98 and a discarded 16-bit index before it.

## The thin items

- **Grenade**, **explosive**, **bolt**, **helmet**, **artefact**, **custom outfit**,
  **detector** — each adds either nothing or one configuration-sourced evaluation-function
  type to the record, and nothing to the payload.
- **Outfit and helmet** each append one quantized condition byte to the *update* only, and
  each declares itself unconditionally network relevant — worn protection is visible on the
  wearer's model.
- **The artefact** is network relevant exactly when it is not in anyone's inventory.
- **The bolt never saves** and never occupies a navigation vertex: it is thrown, it lands, it
  is gone. The predicate returns a bare false with the intended condition commented out
  beside it.
- **The detector** skips its state payload entirely below version 21.
