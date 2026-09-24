# src/xrGame/game_cl_deathmatch_buywnd.cpp

> The buy screen's half of deathmatch: seed it from what the player is actually carrying, apply the rank's weapon upgrades, and send the purchase as a slot-and-item list with a money delta.

**Needs** — [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponKnife.h`](WeaponKnife.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`ui/UIMpTradeWnd.h`](ui/UIMpTradeWnd.h.md) · [`xrServerEntities/xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md)
**Used by** — [`game_cl_deathmatch.h`](game_cl_deathmatch.h.md)
**Tier floor** — T2: builds a purchase over the live inventory and a configuration table

## Purpose

The multiplayer economy as the client runs it. Three decisions carry the file: a purchase is
expressed as a **difference**, not a total; the screen is **seeded from the live inventory**
so that re-buying does not duplicate what you already hold; and a player's rank **substitutes
better weapons for the same default slots** rather than granting extra ones.

## State

`Stateless.` The lists it manipulates belong to
[`game_cl_deathmatch.h`](game_cl_deathmatch.h.md).

## `OnBuyMenu_Ok`

**Contract** — the player confirmed. Flatten the chosen preset into a list of (slot, item)
pairs, force a knife in when the player is waiting to spawn, compute the money delta, and
send it all upstream.

```text
FUNCTION confirm_purchase()
  RETURN IF buying is disabled or there is no controlled object or no local player
  preset = the last-used preset
  IF it is empty AND we are not a living actor THEN preset = the original preset

  items = []
  FOR EACH entry IN preset
    FOR EACH of its count
      resolve the entry's section to (slot, item)
      items.append((attachment flags, item))     # note the packing below

  IF the local player is permanently dead THEN
    items.append(the knife)                      # nobody spawns unarmed

  IF the warm-up is suppressing costs THEN
    write a money delta of 0
  ELSE
    write (cost of the original preset) - (cost of the chosen preset)

  write the item count, then each item's two bytes
  send; if this was reached from a ready press, continue that sequence
  mark the buy screen as awaiting the server's acknowledgement
```

**Invariants**

- **The money delta is original minus chosen**, so it is *positive* when the player spent less
  than he started with — a refund — and negative when he spent more. The server applies the
  delta to the player's balance, so the client never sends an absolute amount and cannot
  simply declare itself rich. It can still lie about the delta; the server re-prices.
- The **knife is unconditional** for a player about to spawn. There is no loadout without one.
- Falling back to the original preset when the chosen one is empty and the player is not alive
  means "spawn with what I had last time" rather than "spawn with nothing", which is what a
  player pressing ready without opening the screen expects.

**Notes** — the entry packs the **attachment flags** into the high byte where the slot would
go, using the same packing helper the slot form uses, and then the wire write sends that byte
as the *slot*. So the receiver reads attachments in the slot field. It works because the
server reconstructs the slot from the item, but the packed record's two field names are lying
at this call site. A rebuild sends (section, attachments) explicitly.

The count is written as a sixteen-bit value and then iterated with an eight-bit counter, so a
preset of more than 255 entries loops forever. Nothing produces one.

## `SetBuyMenuItems` — seeding the screen

**Contract** — fills the buy screen with what the player already has, so the displayed cost is
only for what changes. Two paths.

```text
FUNCTION seed(default_items, only_preset)
  RETURN IF the screen is already open
  reset the screen and begin a player-items block

  IF the local player has a live actor THEN
    defuse every weapon he carries, collecting the freed ammunition
    FOR EACH of his equipped slots, then his belt, then his rucksack
      SKIP an invalid item, a knife, or an item that cannot be traded
      SKIP anything the base-cost table does not price
      SKIP a partially used ammunition box                 # see the invariant
      place it into the screen's matching container, carrying its attachment state
    insert the freed ammunition as additional items
  ELSE
    FOR EACH default item, except the knife
      place it into its slot

  set the screen's money to the player's balance
  end the player-items block and re-check affordability per slot
```

**Invariants**

- **A partially used ammunition box is not seeded.** Only a full box counts as "already owned";
  a half-empty one is left out so the player can buy a fresh one. This rule is repeated at all
  three containers.
- **The knife is excluded everywhere**, because it is granted unconditionally and pricing it
  would double-charge.
- **Defusing first** is what makes a grenade launcher's loaded round reappear as purchasable
  ammunition rather than vanishing with the weapon. The freed ammunition is collected into a
  stack buffer sized at twice the inventory count — twice, because one weapon can free more
  than one kind.

**Notes** — the defusing helper asserts that the player has a live actor *or* is permanently
dead, and then dereferences the actor unconditionally. The caller only reaches it with a live
actor, so it does not fire; the assertion states a condition the code does not actually
tolerate.

## `LoadTeamDefaultPresetItems`

**Contract** — reads a team's comma-separated default loadout from configuration, resolving
each name to a (slot, item) pair and storing it. Names the screen does not know are skipped.

**Invariants** — the stored key is built with a **slot of zero**, discarding the resolved slot.
Every later use re-resolves the slot from the item, so the default list is effectively a list
of items. The commented-out line preserving the real slot sits beside it. A rebuild stores the
item alone.

## `LoadDefItemsForRank`

**Contract** — takes the team's defaults and applies the player's rank to them, then reloads
the screen's default block.

```text
FUNCTION apply_rank()
  defaults = the team's default loadout
  FOR rank IN 1 .. the player's rank            # cumulative, lowest first
    IF there is no configuration section for that rank THEN CONTINUE
    FOR EACH default item
      IF the rank's section names a replacement for that item THEN
        replace it in place

  FOR EACH default item that is not the knife and declares an ammunition class
    take its FIRST ammunition kind
    IF this is not deathmatch THEN append TWO boxes of it to the defaults

  IF the screen is not open THEN
    reset it, open a default-items block, place every default except the knife, close it
```

**Invariants**

- Ranks are applied **cumulatively from one upward**, so a rank-three player receives ranks
  one, two and three in order and the highest applicable replacement wins. A rank with no
  section is simply skipped, which is how the shipped data leaves gaps.
- A replacement is keyed by the *name of the item being replaced*, so the table reads
  "at this rank, whoever would have had X gets Y".
- The free ammunition is **two boxes, and only outside deathmatch**. In deathmatch the loop
  computes the ammunition and then discards it, so deathmatch players receive none. Whether
  that is intended cannot be recovered; the effect is that deathmatch defaults come with an
  empty weapon and the team modes do not.
- Only the *first* ammunition kind a weapon lists is granted. A weapon with armour-piercing
  and standard rounds always gets whichever the configuration lists first.

## `CheckItem`

**Contract** — reconciles one carried item against a preset: place it in its slot, and bring
its three attachments (scope, grenade launcher, silencer) into the state the preset asks for.

```text
FUNCTION reconcile(item, preset, only_preset)
  resolve the item's section to (slot, item); RETURN if unknown
  RETURN if it is a partially used ammunition box
  found = the preset entry for this item, if any
  IF only_preset AND there is none THEN RETURN
  IF the slot is the pistol slot AND this is a default item not in the preset THEN RETURN

  place it in its slot
  desired_attachments = found ? (found's item field shifted right by five) : none
  consume the preset entry

  IF it is a weapon THEN
    FOR EACH of scope, grenade launcher, silencer that the weapon can take
      IF it is currently attached   THEN keep it when the preset wants it, or when
                                          we are not restricted to the preset
      IF it is not currently attached THEN add it when the preset wants it
```

**Invariants** — the "restricted to the preset" flag is what separates two intents: *show me
exactly this loadout* and *show me what I have, adjusted toward this loadout*. Under the first,
an attachment I have but the preset does not want is dropped; under the second it is kept.

The pistol-slot exception stops a rank-granted default pistol from being re-added when the
preset does not mention it, which would otherwise make the default look like a purchase.

**Notes** — the desired attachments are extracted by shifting the entry's **item** field right
by five bits, which only makes sense if that field is holding attachment flags — the same
field confusion as in the confirmation path. The three attachment kinds are a fixed set of
flags shared with the server's weapon record.

## the remaining calls

- **`OnBuyMenu_DefaultItems`** — resets the screen to the player's defaults, restricted to the
  preset.
- **`ChangeItemsCosts`** — empty in deathmatch. The hook exists for modes that price items
  differently per team or per rank; capture the artefact fills it.
- **`OnMoneyChanged`** — while the screen is open, update its balance and re-check which slots
  are affordable. This is what makes a kill reward visible in an open buy screen.
- **`GetBuyMenuItemIndex`** — packs two bytes into the lookup key. The single place the packing
  is written, and it is called with both (slot, item) and (attachments, item).
- **`LocalPlayerCanBuyItem`** — whether a section appears in the screen's groups at all; the
  knife always does. Used by modes that restrict purchases.
- **`AdditionalAmmoInserter`** — places one freed ammunition section into a slot, if the
  base-cost table prices it.

**Notes** — a constant naming an unbuyable slot is defined and never used.
