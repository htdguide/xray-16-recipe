# src/xrGame/ui/UIInventoryUpgradeWnd.cpp

> The upgrade bench: it picks the authored tree shape that matches the item, binds each of that shape's cells to the upgrade the game's registry puts at that coordinate, and turns a click into a confirmed, paid, script-notified installation.

**Needs** — [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) · [`UIInvUpgrade.h`](UIInvUpgrade.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`inventory_upgrade_manager.h`](../inventory_upgrade_manager.h.md) · [`inventory_upgrade.h`](../inventory_upgrade.h.md) · [`Inventory.h`](../Inventory.h.md) · [`Weapon.h`](../Weapon.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`ActorHelmet.h`](../ActorHelmet.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md)
**Tier floor** — T3.

## Purpose

The screen a mechanic opens. Its central idea is a **separation between shape and content**:
the layout document defines a handful of named *schemes* — tree shapes, as columns of cells
at authored positions — and the game's upgrade registry decides which scheme an item uses and
which upgrade sits at each (column, row). Neither knows the other's contents.

That is why an item's upgrade tree can be re-authored in configuration without touching a
layout, and why several items share a tree shape.

## State

```text
RECORD UpgradeBench EXTENDS Window
  schemes         : list<Scheme>           # built once at construction
  current_scheme  : optional<Scheme>
  item            : optional<InventoryItem>  # the subject
  item_picture    : Picture                # its large upgrade-view icon
  item_info       : optional<ItemInfoPanel>
  scheme_pad      : Window                 # cells are attached here
  point_pad       : optional<Window>       # point markers are attached here instead
  repair_button   : Button
  cell_textures   : text per ViewState     # the tables the nodes index
  point_textures  : text per ViewState
  border_texture, ink_texture : text
  pending_upgrade : text                   # the upgrade awaiting confirmation

RECORD Scheme
  name  : text
  cells : list<UpgradeNode>    # owned; each carries its (column, row)
```

**Invariants**

- **Every scheme's cells exist for the whole life of the screen.** Choosing a scheme attaches
  its cells and detaches the previous scheme's; nothing is created or destroyed per item.
- A node's point marker is attached to a **different** parent than the node itself. Markers
  go on a lower pad so the connecting artwork draws behind the cells; nodes go on the scheme
  pad above it. Two parents for one logical widget — the reason the node's hit test must ask
  both.
- The whole screen is optional: a missing layout document makes construction fail and the
  bench is simply absent, the same three-games pattern as the faction-war page.

## Building the schemes

**Contract** — from the layout's template section:

```text
FUNCTION load_schemes(document)
  widescreen := display is wide
  cell_item   := the authored cell rectangle, its WIDTH scaled by 0.8 if widescreen
  cell_border := the authored border rectangle, same scaling, if the document defines one
  FOR EACH template
    scheme.name := the template's name
    FOR EACH column IN template
      FOR EACH cell IN column
        node := new UpgradeNode(bordered = document defines a border)
        node.load(document, column index, cell index, cell_border, cell_item)
        IF the cell authors a point offset THEN build and attach its marker
        append node to scheme.cells
```

**Notes** — the 0.8 width factor is the reciprocal of the 1.2 aspect ratio that appears
everywhere else in this chapter, applied here to *widths* instead of to heights. Heights are
never scaled. It is applied to the shared cell template, so it reaches every cell of every
scheme at once, which is why the individual node does not need to know about it.

The **presence of a border rectangle in the document decides the cells' whole visual idiom**
for the entire screen, not per cell: with one, each cell is a framed box; without, each cell
is an icon with a thin state strip. Both shipped games author one of the two.

## The view-state texture tables

**Contract** — the layout declares a list of cell states, each naming a state by a **fixed
word**, a background texture and a point texture. The words are a closed vocabulary:
`enabled`, `highlight`, `touched`, `selected`, `unknown`, `disabled_parent`,
`disabled_group`, `disabled_money`, `disabled_quest`, `disabled_highlight`. An unrecognised
word is fatal.

**Notes** — the ten words map onto the node's ten view states, and two of them are named
differently in the data than in the code — `highlight` is the focused state and
`disabled_highlight` the disabled-focused one. Those spellings are frozen in the shipped
documents.

Each state also declares an item colour, which is read and then **discarded** — the tint the
node uses is decided by its verdict, in code. Recorded as dead data.

A completeness check over the ten entries exists and is never called; a document missing a
state leaves that state's texture empty and the node renders nothing for it.

## Binding an item

**Contract** —

```text
FUNCTION show_item(cell_item, mechanic_will_upgrade_it)
  fill the item information panel
  item := the inventory item behind the cell item
  dress the large icon: weapon icons from one atlas, outfit and helmet icons from
    another, and the rocket launcher from the outfit atlas despite being a weapon
  detach the previous scheme's cells and markers; disable the repair button
  IF there is no alife simulation or no item THEN RETURN
  enable the repair button when the item's condition is below 0.99
  IF NOT mechanic_will_upgrade_it THEN current_scheme := none; RETURN
  scheme_name := the registry's scheme for this item; none means "no upgrades"
  current_scheme := the scheme by that name          # a missing name is fatal
  FOR EACH node IN current_scheme.cells
    attach the node to the scheme pad and its marker to the point pad
    upgrade := the registry's upgrade at this node's (column, row) for this item
    bind the node to it; dress its five layers from the tables
  recompute every node's verdict
```

**Notes** — the atlas choice is a small type table with one deliberate exception: the rocket
launcher's upgrade icon lives in the outfit atlas because it did not fit in the weapon one.
That is data-driven nowhere and hard-coded here; a rebuild that moves the icon can drop the
exception.

`can_upgrade` comes from outside — it is the mechanic's own willingness, decided in script.
A refusal shows the item and no tree, rather than an empty tree.

The repair button's threshold, 0.99 rather than 1.0, is float slack: an item at full
condition reads slightly under one.

## Installing

**Contract** — three steps, each in a different object:

```text
node click   -> bench.ask("install <name>?", upgrade id)   # remembers the pending id
bench.ask    -> the parent screen puts up a yes/no box
box says yes -> bench.confirm()
                IF the registry installs the upgrade          # this is where money moves
                  notify script "inventory_upgrades.effect_upgrade_item" with the
                    item's script handle and the upgrade id
                  ask the parent screen to refresh the actor and re-separate the
                    upgraded item
                recompute every node's verdict
```

**Notes** — the chapter's event-not-call rule, spelled out over three objects. The node does
not install, the bench does not charge, and the registry does not know a screen exists. The
*only* place the transaction actually happens is the registry call, and everything before it
is asking.

The script notification is fired **after** a successful install and carries the item as a
script handle, so mods can apply side effects — a sound, a reputation change, a statistic —
without the engine knowing about them.

"Re-separate the upgraded item" is the parent screen's job of taking the item back out of any
stack it had been merged into, because an upgraded item is no longer equal to its unupgraded
twins and must stop stacking with them.

## Hover, highlight and delay

**Contract** — setting the currently-described upgrade is filtered twice: the upgrade is
dropped if it belongs to no node of the current scheme, and **dropped again if the node has
not been focused for longer than the information panel's dwell delay**. The surviving value
is handed to the parent screen, which owns the description panel.

**Notes** — the dwell is chapter 15's tooltip rule applied to a panel rather than a tooltip:
sweeping the pointer across a tree must not flash a dozen descriptions. Measuring it from the
node's own focus timestamp — rather than from a timer the bench keeps — is what makes it
correct when the pointer moves between adjacent nodes.

Highlighting a hierarchy is delegated straight to the registry, which marks the upgrades
themselves; the nodes then read their own upgrade's mark each frame. **The highlight lives in
the game data, not in the widgets**, which is how a marker on the lower pad and a cell on the
upper pad stay in agreement without knowing about each other.

Almost every public entry point re-runs the verdict pass over every node first. It is cheap
and it means no caller has to remember to.
