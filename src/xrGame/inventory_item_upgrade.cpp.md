# src/xrGame/inventory_item_upgrade.cpp

> The item's side of the upgrade mechanic: which upgrades it carries, how an upgrade's configuration keys are merged into its tuned numbers, and what must be stripped off the item first.

**Needs** — [`inventory_item.h`](inventory_item.h.md) · [`inventory_item_impl.h`](inventory_item_impl.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`ai_space.h`](ai_space.h.md) · [`Level.h`](Level.h.md) · [`WeaponMagazinedWGrenade.h`](WeaponMagazinedWGrenade.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: configuration merging into live tuned values, plus a network event

## Purpose

An upgrade is a named configuration section whose keys **overwrite** the corresponding keys
of the item's own section on a live item. This file is where that overwrite happens, plus
the membership questions the upgrade forest asks back ("does this item already have X").

It is separate from [`inventory_item.cpp`](inventory_item.cpp.md) purely by subject. The
methods are members of the same carryable mix-in.

## State

The item carries one field for this mechanic: a list of installed upgrade names, in
installation order. It is declared in [`inventory_item.h`](inventory_item.h.md).

**Invariants**
- The list is a *set* — `add_upgrade` refuses a duplicate silently rather than appending.
- The list is **not written to the save file**. It is reconstructed on spawn from the server
  record, which is the authority. That is the reason the whole mechanic runs inside the
  spawn path and not at load time.
- Order is preserved and matters: upgrades are replayed in order on spawn, and two upgrades
  touching the same key resolve last-wins.

## `has_upgrade`

**Contract** — is this upgrade name installed. Answers **yes for the item's own section
name** as well as for anything in the list. That special case is what makes the upgrade
forest's root — which is named after the item's section — count as installed from the start,
so a first-tier group whose parent is the root is immediately available.

## `has_upgrade_group`

**Contract** — is *any* installed upgrade a member of the named group. This is the
prerequisite test the two newer games use (see
[`inventory_upgrade_group.cpp`](inventory_upgrade_group.cpp.md)): the next tier unlocks if
you took *any* of the alternatives at the previous tier.

```text
FUNCTION has_upgrade_group(group_id) -> bool
  FOR EACH name IN upgrades
    IF upgrade_forest.get(name).parent_group_id IS group_id
      RETURN true
  RETURN false
```

**Notes** — this resolves each installed name through the global upgrade forest on every
call, and it is called from inside the forest's own `can_install` walk. Caching the group
name on the item alongside the upgrade name would remove the indirection entirely; nothing
prevents it.

## `add_upgrade`

**Contract** — records an upgrade onto the item and, unless this is a replay during a save
restore, broadcasts an install event so that the authoritative record learns of it too.
Ignores a name already present.

```text
FUNCTION add_upgrade(upgrade_id, loading)
  IF has_upgrade(upgrade_id) THEN RETURN
  upgrades.append(upgrade_id)
  IF NOT loading
    emit event INSTALL_UPGRADE to this object carrying upgrade_id
```

**Notes** — the event goes through the same deferred entity-event path everything else
does, rather than writing the server record directly. The client half may not reach across
and mutate the server object; it asks. In single player both are in one process and the
round trip completes within the frame, which is exactly the confusion the object model
warns about — and exactly why the rule is worth keeping.

## `install_upgrade` / `verify_upgrade`

**Contract** — two entry points into one routine, distinguished by a test flag. `install`
applies the upgrade section's keys to the item; `verify` runs the identical walk and changes
nothing, reporting only whether the section would have had any effect. The screen calls
`verify` to grey out an upgrade that does nothing, and the manager calls `install` to apply
one. Sharing the walk is what guarantees the two cannot disagree.

## `install_upgrade_impl`

**Contract** — merges an upgrade section into this item's live tuned values. Each key is
optional: a key absent from the upgrade section leaves the item's value alone. Reports
whether *any* key was present. When the test flag is set, reads and reports but does not
assign.

```text
FUNCTION install_upgrade_impl(section, test) -> bool
  touched  = merge(section."cost"       -> cost)
  touched |= merge(section."inv_weight" -> weight)

  IF this item has a base slot                     # only slotted items have handling
    touched |= merge(section."default_to_ruck"  -> flag: goes to the backpack by default)
    touched |= merge(section."sprint_allowed"   -> flag: may sprint while holding this)
    IF NOT normalize_upgrade_sensitivity
      touched |= merge(section."control_inertion_factor" -> aim inertia)
    ELSE IF normalize_sensitivity AND aim inertia is still unset
      raw = section."control_inertion_factor", default 1
      # compress the authored factor toward 1 before using it as an offset
      IF |raw| > 1     THEN raw = raw / 4
      ELSE IF |raw| >= 0.5 THEN raw = raw / 3
      ELSE IF |raw| > 0.1  THEN raw = raw / 2
      aim inertia = clamp(1 + raw, 0.1, 1)

  touched |= merge(section."immunities_sect"     -> replace the damage-immunity table)
  touched |= merge(section."immunities_sect_add" -> add to the damage-immunity table)
  RETURN touched
```

**Invariants** — the handling keys are read only for an item with a base slot. An item with
no slot is never in the hands, so a sprint permission or an aim-inertia factor on it would
be meaningless; reading them would also mean an upgrade could set a handling value that
nothing ever consults.

**Notes** — the mouse-sensitivity branch is a compatibility repair, not a design. The
shipped data authors `control_inertion_factor` as a value the original engine used one way
and modifications used another, and two console flags select which reading is in force. The
divide-by-four/three/two ladder has no derivation in the source: it is a hand-tuned
compression of the authored range into something usable as an offset from 1. A rebuild
should treat the ladder as data, not as a formula — and treat the whole branch as optional,
since it is off unless the player turns the normalization flag on.

Two immunity keys exist because an upgrade may either *replace* the item's damage-immunity
table wholesale or *add* to it, and the shipped outfit upgrades use both.

## `pre_install_upgrade`

**Contract** — puts the item into a state where its numbers may safely be rewritten. Run
immediately before an install. Nothing here is about upgrades as such: it is about the item
having *live* state derived from the numbers that are about to change.

```text
FUNCTION pre_install_upgrade()
  IF this is a magazine weapon
    unload the magazine
    IF it also has an attached grenade launcher
      switch to the launcher, unload it too, switch back
  IF this is a weapon
    detach the scope, the silencer and the grenade launcher if attached
```

**Notes** — the magazine and the attachments are separate objects whose existence depends on
the weapon's configuration. An upgrade may change the ammunition type, the magazine size or
which attachments are permitted; rounds and attachments that were valid before must be
returned to the owner's inventory first, or they are silently destroyed. Switching to the
grenade launcher and back to unload it is the only way to reach its magazine — the weapon
exposes exactly one loaded magazine at a time, whichever barrel is selected. That is a
consequence of the weapon's own design, and a rebuild that models two magazines directly
does not need the round trip.

## `get_upgrades_str` / `equal_upgrades` / `log_upgrades`

**Contract** — `get_upgrades_str` renders the installed set as a comma-joined list of the
*configuration sections* the upgrades name, not their identifiers, for the screen's tooltip;
it reports whether anything was written. `equal_upgrades` compares two upgrade sets
disregarding order, which is what the trade and repair screens use to decide whether two
otherwise identical items are interchangeable. `log_upgrades` is a development-build dump.

**Notes** — `equal_upgrades` is a quadratic set comparison over lists that are at most a
handful of entries. That is a reasonable choice at this size and a bad one if a rebuild
lets the list grow.

## `net_Spawn_install_upgrades`

**Contract** — reconstructs the item's upgrade state as the object comes to life. Runs only
in single player with the alife simulation present — multiplayer has no upgrade mechanic —
and only if the server record is an item record at all.

```text
FUNCTION net_Spawn_install_upgrades(server_record)
  IF NOT single_player OR NO alife simulation THEN RETURN
  IF server_record is not an inventory-item record THEN RETURN

  saved = server_record.upgrades        # copy: the next step clears the live list
  upgrades.clear()
  upgrade_forest.init_install(self)     # apply this section's own base upgrade state
  FOR EACH name IN saved
    upgrade_forest.upgrade_install(self, name, loading = true)
```

**Invariants** — the saved list must be copied before the live list is cleared, because the
two may be the same storage when an object is respawned in place.

**Notes** — this is the answer to "where do upgrades live". Not in the save file's item
chunk; in the server record, replayed through the forest on every spawn. The replay runs
with the loading flag set, which as
[`inventory_upgrade_base.cpp`](inventory_upgrade_base.cpp.md) describes skips every policy
check and hardens every consistency check — so a record that violates the forest's structural
rules fails loudly here rather than producing a quietly different item.

The consequence a rebuild must accept: **the forest is part of the save format**. Changing
which group an upgrade belongs to invalidates existing saves, because the replay will refuse
a combination that was legal when it was written.
