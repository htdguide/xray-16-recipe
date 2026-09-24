# src/xrGame/inventory_upgrade_manager.cpp

> Builds every item's upgrade tree from configuration at startup, and is the single place an upgrade is checked, installed and recorded onto an item.

**Needs** — [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md) · [`inventory_upgrade.h`](inventory_upgrade.h.md) · [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md) · [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md) · [`inventory_upgrade_property.h`](inventory_upgrade_property.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: parses configuration into an object graph and walks it

## Purpose

Weapon and outfit upgrading is a mechanic where a technician permanently rewrites an
item's tuned numbers in exchange for money. The authored data is a forest: one **root** per
upgradeable item section, each root holding a chain of **groups**, each group holding
mutually exclusive **upgrades**, each upgrade naming a configuration section whose keys are
merged into the item when it installs. This file owns that whole forest — it constructs it,
it is the only owner of every node, and every operation the game or a script performs on an
upgrade passes through it.

It is a separate file from the node types because the nodes are mutually recursive: a root
asks the manager to create its groups, a group asks for its upgrades, an upgrade asks for
the group it unlocks. Only a third party holding all four tables can close that cycle.

## State

```text
RECORD UpgradeManager
  roots      : map<text, Root>       # keyed by the ITEM's section name
  groups     : map<text, Group>
  upgrades   : map<text, Upgrade>
  properties : map<text, Property>   # public: the upgrade screen iterates it
```

**Invariant** — the manager owns every node in all four tables; the parent and child links
between nodes are borrowed references. Destruction frees the tables, never the links, so
there is no order dependency between them at teardown.

**Invariant** — identifiers are globally unique across the whole forest, not per-item: two
different items may not declare upgrades of the same name, and an upgrade is found by name
alone without knowing which item it belongs to. The install path exploits this — it looks
the upgrade up globally and only *diagnoses* whether the item's root actually contains it.

**Invariant** — construction order is properties first, then the item forest. Upgrade
nodes reference properties by name as they construct, so the property table must already
be complete. This is the file's one hard ordering constraint and it is marked as such in
the original.

**Invariant** — the tables are keyed by the *identity* of the interned string rather than
its text. Interned strings are unique per distinct text, so ordering by identity is a
valid total order and cheaper than comparing characters — but it is not *stable*, so no
part of the game may depend on the iteration order of these tables. Diagnostic listings
are the only iteration and they are unordered.

## `Manager` construction

**Contract** — reads all upgrade data at construction, with no arguments: properties from
the fixed section `upgrades_properties`, then one root for every configuration section in
the entire game that declares both an `upgrades` key and an `upgrade_scheme` key. Both
lines must be present and non-empty, and a section missing either is silently not
upgradeable. Blocks; performs the bulk of its allocation here.

**Notes** — scanning every section in the game to discover upgradeable items is a
deliberate change from the original games, which listed them in a dedicated section. The
scan is what lets a mod add an upgradeable item without editing a central list, at the cost
of one pass over the whole configuration at startup. Keep the scan; the list form is a
strictly smaller behaviour.

```text
FUNCTION load_all_properties()
  section = "upgrades_properties"
  IF section is absent OR empty THEN log and RETURN   # upgrades still work, unlabelled
  FOR EACH key IN section
    add_property(key)                                 # the KEY names a property section

FUNCTION load_all_inventory()
  FOR EACH section IN every configuration section
    IF section has a non-empty "upgrades" AND a non-empty "upgrade_scheme"
      add_root(section)
```

## `add_root` · `add_group` · `add_upgrade` · `add_property`

**Contract** — each allocates a node, **inserts it into its table, and only then asks the
node to construct itself**. That order is load-bearing: a node's construction recurses into
the manager to create its children, and a child may name the node being constructed as its
parent. Inserting last would let that recursion create a duplicate.

`add_group` is the one that differs. A group can be reached from more than one parent
upgrade — that is how two separate upgrade branches converge on one shared group — so
asking for an existing group **attaches the new parent to it and returns the existing
node** rather than creating a second. The other three treat a duplicate name as an authoring
error, log it in a non-shipping build, and then overwrite the table entry anyway, leaking
the previous node. A rebuild should reject the duplicate instead.

## `upgrade_install`

**Contract** — the real install. Resolves the upgrade by name, asks it whether it may be
installed against this item, and on approval performs the three steps below in order.
Returns whether the install happened. A resolvable upgrade whose configuration section
turns out to be empty is a hard failure, not a refusal: an authored upgrade that changes
nothing is a data bug the player would otherwise pay for and not notice.

The `loading` flag distinguishes replaying a saved or authored upgrade from a player buying
one. When loading, the item's pre-install hook is skipped (it exists to snapshot state for
the purchase animation and the money transaction) and the upgrade's scripted effects are
told they are being replayed, so that a one-shot effect — a message, a reputation change —
does not fire again on every load.

```text
FUNCTION upgrade_install(item, upgrade_id, loading) -> bool
  upgrade = resolve(item.section, upgrade_id)   # diagnoses mismatch, still returns the node
  IF upgrade.can_install(item, loading) IS NOT ok THEN RETURN false

  IF NOT loading THEN item.pre_install_upgrade()

  IF NOT item.install_upgrade(upgrade.section) THEN
    FAIL WITH "upgrade section is empty"        # data bug, never a runtime condition

  upgrade.run_effects(loading)                  # scripted side effects
  item.add_upgrade(upgrade_id, loading)         # records it on the item, so it saves
  RETURN true
```

**Invariant** — the ordering is: numbers merged into the item, then effects, then the
upgrade recorded on the item. Effects run after the merge because an effect may read the
item's new values; the record is written last so that a failed merge leaves no trace.

## `upgrade_add`

**Contract** — the same install with the checks and the effects removed: it merges the
section and records the upgrade, but runs no effects and never calls the pre-install hook.
This is the path used when an item arrives already upgraded — a trader's stock, a scripted
reward — where the upgrade is a property of the item rather than an event that happened.
It uses a separate permission check (`can_add`) which does not require the player to have
unlocked the prerequisites.

## `init_install`

**Contract** — applies an item's authored `installed_upgrades` list at spawn, in list
order, each as a loading install. This is how an item ships pre-upgraded from data. A
non-upgradeable item returns immediately. Does nothing about upgrades recorded in a save —
those are replayed by the item's own load path.

**Invariant** — list order is the install order, and the later entries may depend on the
earlier ones having been merged, so it may not be reordered or parallelized.

## `can_install_upgrade` · `can_add_upgrade`

**Contract** — the dry runs behind the two install paths, each returning only whether the
corresponding operation would be permitted. They exist so the upgrade screen can grey out a
cell without attempting it.

## `make_known_upgrade` · `is_known_upgrade`

**Contract** — get and set the per-upgrade *known* flag: whether the player has discovered
this upgrade exists. Each comes in two forms — one taking an item, which additionally
diagnoses whether the upgrade really belongs to that item's tree, and one taking only the
upgrade name, which is what scripts call. The flag lives on the upgrade node, which means
it is **global, not per-item-instance**: teaching one upgrade reveals it on every copy of
that item in the world. That is the intended mechanic — knowledge belongs to the player.

**Notes** — the item-scoped forms dereference the resolved node without checking it, so a
name absent from the table is a crash rather than a diagnostic there, while the bare forms
check. The asymmetry is not deliberate.

## `upgrade_verify`

**Contract** — resolves an upgrade by name and, in a non-shipping build, reports three
distinct authoring mistakes: the item has no upgrade tree, the upgrade name does not exist,
and the upgrade exists but under a different item. It **returns the node regardless** —
verification is diagnosis, not a gate. A rebuild that turns these into refusals changes
behaviour, because the shipped data relies on cross-item upgrade names resolving.

## `compute_range`

**Contract** — finds the smallest and largest value one named item parameter takes across
every upgradeable item section *and* every upgrade section in the game, so the upgrade
screen can draw a parameter as a filled bar on a common scale. Returns whether any value
was found at all; a parameter nothing declares yields no range and the screen falls back to
a plain number.

```text
FUNCTION compute_range(parameter) -> optional<(low, high)>
  low = +infinity ; high = -infinity
  FOR EACH section IN (every root's item section) THEN (every upgrade's section)
    IF section declares parameter AND its value is non-empty
      low = min(low, value) ; high = max(high, value)
  RETURN none IF nothing was seen
```

**Notes** — the scan is over the whole forest every time it is asked, and the screen asks
once per displayed parameter. It is small enough not to matter; a rebuild may memoize it,
since the configuration does not change at run time.

## `get_item_scheme` · `get_upgrade_by_index`

**Contract** — the upgrade screen's two layout lookups: the name of the layout scheme
authored for an item, and the upgrade occupying a given cell of that scheme. Both return
nothing for a non-upgradeable item; an empty cell is reported as a diagnostic because the
screen only asks about cells the scheme declares.

## `highlight_hierarchy` · `reset_highlight`

**Contract** — lights the path through the tree that leads to one named upgrade, and clears
all lighting. Purely presentational state living on the nodes; see
[`inventory_upgrade_root.cpp`](inventory_upgrade_root.cpp.md) for what "the path" means,
which differs between games.

## `item_upgrades_exist`

**Contract** — the predicate the startup scan uses: a section is upgradeable when it exists
and declares both a non-empty `upgrades` list and a non-empty `upgrade_scheme`. A section
name that does not exist is a caller bug and is reported, but yields a plain negative rather
than aborting, so that one bad reference cannot prevent the game from starting.
