# src/xrGame/ui — chapter 25

> The game's actual screens. Every one is a tree of chapter-15 widgets, declared in shipped
> data and found by name at construction, raised onto a dialog stack that takes the keyboard
> away from the world, and driven by polling the simulation rather than by listening to it.

## What this module is responsible for

Everything the player interacts with that is not the world itself: the inventory, the trade
counter, the upgrade bench, the loot panel, the personal terminal with its map and task list,
the conversation screen, the heads-up overlay, the tutorial and loading screens, the main
menu's spinner, and the two dozen multiplayer screens.

It is not the widget toolkit. Buttons, lists, text and the event model are
[chapter 15](../../xrUICore/README.md); this chapter is what is built out of them. The
division is clean in one direction and leaky in the other: the toolkit knows nothing about the
game, while nearly every screen here reaches directly into the simulation — an actor's
condition, an inventory, a relation registry, a trade handle — and reads it every frame.

## Where it sits

Inside the game module, and therefore after the engine, the toolkit and the script layer. It
rests on the toolkit for every widget and for the XML reader; on the engine for the frame
loop, the input pump, the console and the localization table; on the script engine, which both
drives screens and is driven by them; and on almost every part of the game layer, because a
screen's job is to show the game.

Two parts of the chapter reach seams directly: the multiplayer screens reach
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
and [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport);
the menu's sound and the tutorial's video reach
[Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) and
[Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs).

## The load-bearing ideas

Named once here, so the two hundred pages below can be terse.

### A screen is raised onto a stack, and only the top of it hears the keyboard

A **dialog holder** owns the screens. Exactly one holder is active: the in-game UI while a
level runs, the main menu otherwise. It keeps **two separate collections**, and conflating
them is the classic rebuild mistake:

- an **input stack** — only its top element receives keyboard, pointer and controller events,
  and only its `NeedCursor` decides whether a cursor is visible;
- a flat **render list**, in raise order — every enabled entry draws, so a screen can be
  visible without being interactive.

Raising a screen is one ordered act: assert it is not already shown (double-raise is a hard
error), add it to the render list and show it, push it onto the input stack, **snapshot the
crosshair and indicator state into that stack entry**, bind the holder, show the cursor if the
screen wants one, and — if a level is loaded — freeze the actor when the screen says to and
synthesise a release of the fire and zoom keys so the player is not left shooting under the
screen.

Dismissing reverses it, with one asymmetry worth knowing: a screen that is *not* the top of
the input stack is erased from the middle, and that path does not restore the indicator
snapshot. Removal from the render list is **deferred** — the entry is disabled, and the list
is compacted at the end of the next frame — so that a screen may close itself from inside its
own draw or update.

### Almost nothing pauses the simulation; the inventory dilates time instead

Raising a screen does not pause. Only three things pause: the pause key, the main menu, and a
tutorial or video (each of which snapshots the prior pause state and restores it).

The in-game screens take a different route: opening the inventory or the personal terminal
sets a **time dilation factor**, and closing it restores unity. The world keeps running,
slowly. That has a consequence throughout the chapter — every timeout a screen measures
against the clock is scaled by the dilation factor, or it would feel several times longer with
a screen open than without. The tooltip dwell in the inventory does this; so does the map's
hover hint.

The main menu is the one holder that sets **ignore-pause**, so its screens keep receiving
input while the engine is paused. Individual screens can opt in the same way — the level
change prompt and the demo playback control both do.

### Input reaching the world is a property of the screen, not of the holder

A screen declares whether it **swallows movement**. When it does, unconsumed input stops at
the screen; when it does not, the binding is forwarded on to the world and the player can keep
walking. The inventory declares *no* in inventory mode and *yes* in trade, upgrade and loot
modes. The personal terminal and the speech menu declare no; everything else declares yes.

Four key bindings are swallowed unconditionally regardless — the four quick-use slots — so
that a screen can never fire them twice.

A second, separate channel exists for controllers: when the cursor is visible, the four
directional UI actions are converted into a **geometric focus move** through the toolkit's
navigation registry (chapter 15), and the analogue stick drives the cursor through an
acceleration ramp with a configurable sensitivity. The cursor auto-hides after an idle
interval, so a controller player is not left with a pointer parked on screen.

### Every screen is declared in shipped data, and the names are frozen

A screen loads one XML document and then finds each of its widgets by a **colon-separated
element path plus an index**: `minimap:static_counter:text_static`, `dragdrop_bag`,
`actor_ch_info`. There is no registry, no reflection and no late binding — the lookup happens
once, at construction, and the resulting pointer is kept.

That makes **every element name in every shipped layout part of the interface**. Renaming an
element is renaming a symbol.

One helper family performs the whole operation — allocate the widget, configure it from the
element, attach it to a parent, mark it owned by that parent — and carries the chapter's
entire error policy in a single flag:

- **required and missing** → a hard failure naming the element path and the document. A
  half-built screen is worse than a stop.
- **optional and missing** → no widget at all, and every later use guards.

Optional is not the exception. Three games ship three differently authored layouts for several
of the same screens, so a large fraction of every screen's widget set is optional and a
rebuild that assumes a complete set will crash on at least one shipped game. Two of the
create helpers do not honour the optional flag and crash anyway; that is a defect the source
itself flags.

Alongside the named lookups, a document's `auto_static` children are created and attached
**wholesale with no code-side names** — which is how decorative statics get on screen without
a line of code per widget.

Text is never a literal: an element's text content is an identifier into the localization
table. Colours come from a named table loaded from data, and an unknown colour name is fatal.

A **style** mechanism (chapter 15) re-points the path prefixes all of this goes through, so
the same code loads a different skin's documents without knowing it.

### The inventory screen is one window with four modes

The inventory, the trade counter, the upgrade bench and the loot panel are one screen wearing
four faces. A mode change runs an exit step for the old mode and an entry step for the new,
always in that order, because the two touch overlapping widgets — in one layout dialect the
same widget *is* two different lists in two different modes.

Sixteen cell lists live in one array. Every drop rule, every move rule and every highlight
rule is written not against a list's identity but against its **role** — actor slot, actor
bag, actor belt, actor's trade side, partner's trade side, partner's bag, loot bag, quick
slot, trash — and the mapping from widget to role is many-to-one and mode-dependent. That
indirection is the whole reason one handler set can serve four screens, and it is exported to
script, so the role names are frozen.

**Which drops are legal is one table**, filled at construction: for each destination role, the
set of source roles it accepts. Read it for what it forbids — the two trade sides are sealed
from each other, nothing may be dropped into a partner's bag, and the trash accepts only from
places the actor owns.

### Drag and drop, stacking, and splitting

A cell list is a grid of fixed-size cells. An item declares a **footprint in cells**, and the
same two numbers are simultaneously its sub-rectangle in the shared 50-unit icon atlas — so an
item's icon can never be a different shape from the space it occupies. Placing an item writes
the same widget into every covered cell, with a flag marking the anchor, so hit-testing any
covered cell finds the item.

There is **exactly one drag in flight in the whole process**, held in a single shared slot,
because there is exactly one pointer. The floating widget under the cursor registers itself
directly with the frame loop below every other UI consumer, so it draws over every screen
including the one that owns it. It remembers the offset from the cursor to the picked-up
corner and re-derives its position from the cursor every frame rather than accumulating
deltas — which is what makes grabbing an icon by its corner feel different from grabbing it by
its middle. The **drop target is resolved by hover**, each list claiming the drag as the
cursor enters it, not by the drop event.

Identical items **stack**: one cell becomes the head and the others its children, exactly one
level deep. Mergeability is a chain of narrowing tests — the same footprint, then the same
section, then a condition within one percent, then the same upgrades, then for a weapon the
same attachments and the same scope. The one-percent tolerance is a gameplay decision: exact
equality leaves an inventory littered with near-identical singletons.

Splitting is where a rebuild goes wrong. Popping from a stack always removes the **last**
child widget and then **swaps payloads** so the caller receives the item it asked for. Widgets
are recycled; the item's identity is its payload, not its widget. That is what keeps a stack
head in its grid position with its highlights intact across any number of splits, and it means
any code holding a widget across a split is holding a widget whose item has changed.

Items are packed into the grid greedily in insertion order, so every fill sorts by
**descending footprint first**. Drop the sort and a full inventory stops fitting in the same
grid it fitted in before.

### A UI action is an event, never a call

This is the single most important decision in the chapter. When the player drags a magazine
into a slot, the screen moves its own widgets immediately **and separately emits a packed game
event** describing the intent, addressed to an entity and applied by the authoritative side.
The screen does not wait and does not reconcile.

That is what makes one set of screens serve both single player — where both sides are the same
process and the effect is immediate — and multiplayer, where the screen is optimistic and the
server may disagree. Six such senders exist: item to slot, to belt, to bag, use, drop, and
activate slot. An ownership transfer between two inventories is expressed as a **sell event
followed by a buy event**, in that order, even when no trade is happening and no money moves,
because that pair is the only ownership move the entity event system knows.

The reverse direction is a single notification — "this item was acquired" or "this item was
lost" — and handling it correctly means searching *every* list and removing the item from all
but the one it now belongs in, because an optimistic UI's classic failure is the duplicate.

### The overlay polls, and it polls slowly

The heads-up display — health, stamina, armour, ammunition, the minimap, contacts, the
condition and booster icons, the fire-mode and grenade indicators — is not on the dialog
stack. It is drawn and updated directly by the in-game UI, under a guard that hides it when
indicators are off.

Its update is **throttled to one frame in ten**. Cheap work — the minimap, the motion icon,
the pick-up preview — runs every frame; everything else, including all the condition
indicators, runs on the tenth. The minimap's contact counter is slower still, one frame in
twenty. Drawing is never throttled. A rebuild is free to choose different intervals but should
know that the numbers are visible: indicator icons appear up to a sixth of a second late, and
that was judged acceptable in 2007.

**The hit direction indicator is not part of the overlay.** It is a separate marker list owned
by the head-up manager. A hit pushes a marker recording the *negated* hit direction — a
heading only, pitch discarded — and the clock. The marker's fade is driven entirely by a
named authored colour animation rather than by a hardcoded duration: it is alive while the
elapsed time is under the animation's length, and its alpha is the animation's alpha. It is
drawn rotated by the camera's heading plus its own, so marks are world-locked and swing as the
player turns. Grenade markers reuse the same animation at double speed and are *refreshed*
each frame while the grenade lives rather than expiring.

The anomaly sensing that drives the detector click and the sensor needle is a **peak-hold with
exponential decay**: a value only ever rises to meet a stronger reading and decays by a fixed
factor per frame, clamped to zero below a small threshold. The inventory screen's condition
panel reads the *same* value rather than computing its own, so that the two agree.

Progress bars quantise to the number of segments their artwork was drawn with — 55 for health,
31 for the protection bars, 15 for a list's condition indicator, 13 for a cell's. Those
numbers are frozen alongside the textures.

### The personal terminal is a tab control over six independent pages

Tabs are named by string, and a named-section table maps each to a page: the map, the tasks,
the faction war, the player's statistics, the ranking and the logs. Every page is constructed
only in single player, and **a page whose layout document is missing is simply deleted** — a
null page is a legitimate state throughout, not an error.

Switching a tab detaches the previous page, then offers the section name to a script function
that may **override** the engine's choice with a script-built window, then attaches and
captures the keyboard. Every switch also publishes an information portion naming the section,
which is how the game's own scripts learn the terminal was opened on a particular page.

Only the active page updates; the others are frozen. One page — the logs — is explicitly
scheduled onto the engine's parallel work list rather than updated inline, because building
its rows is expensive.

**The map is one zoom continuum, not two screens.** A global map holds one child per level,
each level's rectangle within the global map coming from configuration, and each level map
re-derives its own rectangle every frame by projecting that rectangle through the global map's
current transform. At minimum zoom a level map shows a hover hint with its name instead of
showing spots.

**A spot is a model object, not a widget.** It owns up to three widget representations — one
for the level map, one for the minimap, one composite with icons and a countdown — plus an
off-screen pointer arrow and border decorations, and it caches its resolved position per
frame. The spot list is owned by the map manager, keyed by (type, entity), and the *visual*
definition of each type comes from a resident layout document kept loaded for the whole
session so spots can be built lazily.

Neither map registers as a listener. Each frame the map **detaches all of its children and
rebuilds them** from the spot list. The level map culls first — skipping entirely when its
rectangle does not intersect the visible area — while the minimap does not cull at all.

Map navigation is driven by a **goal planner**, the same planner the creature AI uses: setting
a target map and centre gives the planner a goal and it animates the transition. The
navigation buttons are a fixed nine-slot pad. Filters are four hardcoded categories —
treasures, quest characters, secondary tasks, primary objectives — and the legend is a pure
display panel with no binding to the live spot list.

### The dialogue screen drives a phrase graph, and the character speaks first

The conversation is a **graph of phrases** keyed by string identifier, with a precondition on
every edge. The screen owns the whole of the state; the view beneath it is a dumb pair of
scrolling lists that reports back through three notifications and a single one-slot mailbox
holding the clicked phrase's identifier.

One value is the entire mode switch: the current dialog handle. Absent means "list the
conversation topics the player may open"; present means "we are inside a graph".

**The other character has the initiative.** On opening, the screen asks the partner for its
available dialogs, takes the highest-priority one, and immediately speaks phrase `"0"` on the
partner's behalf. Only if the partner has nothing to say does the conversation start in topic
mode.

Saying a phrase **toggles a single boolean** that says whose turn it is, then rebuilds the
list of admissible successors from the graph's out-edges, admitting each only if its script
precondition passes. No admissible successor means the conversation is finished. The player's
question list is populated only when it is the player's turn; when it is the partner's, the
list stays empty and the screen waits.

Two special cases matter. A list of successors that are all placeholders is **auto-spoken** —
one chosen at random, with no player action — which is how a conversation runs a scripted
passage. And a phrase with no text is dropped from both the log and the list entirely.

Voice lines are found by **convention**: a fixed directory prefix plus the phrase's own
localization key. The partner's lip-sync handler gets first refusal; failing that the screen
plays it positioned at the partner.

Trade is launched from the conversation by hiding the *view* while leaving the screen on the
input stack, so returning from trade brings the conversation back exactly as it was. A
"cannot break off" flag suppresses both the exit button and the escape binding, which is how a
scripted conversation is made unskippable.

### Most multiplayer screens reach a service that no longer exists

Roughly a third of this directory is multiplayer: the server browser, the buy menu, the
scoreboard, the voting and administration dialogs, the spawn and skin screens, the chat line.
The matchmaking, account and server-list service they depend on is
[given, and dead](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) —
shut down in 2014, with no compatible substitute.

The vendor wrapper is nonetheless an unconditional part of the build with no opt-out, and the
code is honest about the outcome: the availability check spins on a handshake and reports that
the online services are no longer available. So the browser's whole query, filter and sort
path is reachable and the list is permanently empty.

Two consequences for a rebuild. First, treat this as an **optional module with a null
implementation**; a rebuild that wants multiplayer is designing a fresh master-server
protocol, not restoring this one. Second, the browser's detail pane is worth reading as a
*specification* of what such a protocol must answer — it enumerates, per game type, every
field the original service reported.

Separately, the **transport** seam ships its null filling by default, so even a local
multiplayer game needs the build description edited before anything connects.

One pattern recurs across every multiplayer dialog and is worth stating once: a vote or
administration dialog collects a choice, formats it into a **console command string**,
executes that string, and closes. It never participates in the vote. A rebuild may replace the
console channel with anything; what matters is keeping the separation.

### Two screens are settings controls, and one uses a store outside the game

The product-key and player-name fields implement chapter 15's four-step settings protocol
(read, back up, commit, undo) against a **per-machine settings store** rather than a console
variable — the one place that protocol is used for something that is not a console variable.
That store sits behind
[Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services);
the original reaches it through a platform-specific path, and on platforms without one the
player's name falls back to the operating system's own account record.

### The script surface is frozen

Several screens are exported to Lua by class name, with their method names, their enumerations
and their expected script callbacks fixed by conformance criterion 10. Beyond the exported
classes, the engine **calls into script by name** in a dozen places — to fill a faction's war
state, to decide whether an item may be repaired or upgraded and to write the sentence the
player is shown, to label and to gate a context-menu action, to override a terminal page, to
react to an item being dropped on another item. Every one of those names is part of the
interface.

## Twins

Grouped by what they build, not alphabetically.

### Shared plumbing

| Twin | Role |
|---|---|
| [`UIDialogWnd.cpp`](UIDialogWnd.cpp.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) | The base every stacked screen derives from: the holder link, the pause gate, and raise/dismiss |
| [`UIXmlInit.cpp`](UIXmlInit.cpp.md) · [`UIXmlInit.h`](UIXmlInit.h.md) | The game's four additions to the toolkit's layout reader: cell lists, multiplayer tabs, the sleep static, the hint window |
| [`UIHelper.cpp`](UIHelper.cpp.md) · [`UIHelper.h`](UIHelper.h.md) | Create-configure-attach, and the required-versus-optional element policy for the whole chapter |
| [`UIInventoryUtilities.cpp`](UIInventoryUtilities.cpp.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) | The shared icon atlas, the game clock's display forms, and the rank, reputation and goodwill bands |
| [`UIColorAnimatorWrapper.cpp`](UIColorAnimatorWrapper.cpp.md) · [`UIColorAnimatorWrapper.h`](UIColorAnimatorWrapper.h.md) | Playing an authored colour curve at UI update rates, against a wall clock rather than the frame delta |
| [`UIFrameLine.cpp`](UIFrameLine.cpp.md) · [`UIFrameLine.h`](UIFrameLine.h.md) | The game's own stretchable three-segment line, beside the toolkit's |
| [`UILabel.cpp`](UILabel.cpp.md) · [`UILabel.h`](UILabel.h.md) | A static that sizes itself to its text |
| [`UIStatix.cpp`](UIStatix.cpp.md) · [`UIStatix.h`](UIStatix.h.md) | A picture that fits itself into an authored box |
| [`UITextBanner.cpp`](UITextBanner.cpp.md) · [`UITextBanner.h`](UITextBanner.h.md) | The animated text banner |
| [`UISleepStatic.cpp`](UISleepStatic.cpp.md) · [`UISleepStatic.h`](UISleepStatic.h.md) | The sleep transition's progress display |
| [`UIListItemAdv.cpp`](UIListItemAdv.cpp.md) · [`UIListItemAdv.h`](UIListItemAdv.h.md) | A list row with an icon and two text fields |
| [`UIMessageBoxEx.cpp`](UIMessageBoxEx.cpp.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) | The toolkit's message box wrapped as a stacked dialog with yes/no callbacks |
| [`UIWheelMenu.cpp`](UIWheelMenu.cpp.md) · [`UIWheelMenu.h`](UIWheelMenu.h.md) | The radial selection menu |
| [`UIScriptWnd.cpp`](UIScriptWnd.cpp.md) · [`UIScriptWnd.h`](UIScriptWnd.h.md) · [`UIScriptWnd_script.cpp`](UIScriptWnd_script.cpp.md) | The base a Lua-authored screen derives from, and its frozen export |
| [`UIDebugFonts.cpp`](UIDebugFonts.cpp.md) · [`UIDebugFonts.h`](UIDebugFonts.h.md) | A development page showing every font in the set |
| [`CExtraContentFilter.cpp`](CExtraContentFilter.cpp.md) · [`CExtraContentFilter.h`](CExtraContentFilter.h.md) | Whether a piece of bonus-edition content is unlocked on this machine |

### Key binding, loading and tutorials

| Twin | Role |
|---|---|
| [`UIKeyBinding.cpp`](UIKeyBinding.cpp.md) · [`UIKeyBinding.h`](UIKeyBinding.h.md) | The key-binding page: the action list and its two columns |
| [`UIEditKeyBind.cpp`](UIEditKeyBind.cpp.md) · [`UIEditKeyBind.h`](UIEditKeyBind.h.md) | One binding cell, which captures the next key pressed |
| [`UILoadingScreen.cpp`](UILoadingScreen.cpp.md) · [`UILoadingScreen.h`](UILoadingScreen.h.md) | The level loading screen: the stage text, the bar, and the tip |
| [`UILoadingScreenHardcoded.h`](UILoadingScreenHardcoded.h.md) | The fallback loading screen used when no layout document can be read |
| [`UIGameTutorial.cpp`](UIGameTutorial.cpp.md) · [`UIGameTutorial.h`](UIGameTutorial.h.md) | The scripted tutorial sequence: timed items, pause capture, input blocking |
| [`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md) | A tutorial step that shows statics and plays a sound |
| [`UIGameTutorialVideoItem.cpp`](UIGameTutorialVideoItem.cpp.md) | A tutorial step that plays a video |
| [`MMSound.cpp`](MMSound.cpp.md) · [`MMSound.h`](MMSound.h.md) | The main menu's two sound channels: the spinner and the music shuffle |
| [`UIMMShniaga.cpp`](UIMMShniaga.cpp.md) · [`UIMMShniaga.h`](UIMMShniaga.h.md) | The main menu's rotating item spinner |

### The inventory screen and its four modes

| Twin | Role |
|---|---|
| [`UIActorMenu.h`](UIActorMenu.h.md) | The screen's declaration, the nine list roles and the four modes; a map of the eight files below |
| [`UIActorMenu.cpp`](UIActorMenu.cpp.md) | The mode machine, the per-frame poll, the widget-to-role mapping, the selection, and the highlight rules |
| [`UIActorMenuInitialize.cpp`](UIActorMenuInitialize.cpp.md) | Construction from either layout dialect, the list table, and the drop-permission table |
| [`UIActorMenuInventory.cpp`](UIActorMenuInventory.cpp.md) | Filling the lists, every move operation, the context menu, and how a UI action becomes a game event |
| [`UIActorMenu_action.cpp`](UIActorMenu_action.cpp.md) | Every gesture — drop, double click, right click, focus, keys — routed by list role |
| [`UIActorMenuTrade.cpp`](UIActorMenuTrade.cpp.md) | Trade mode: staging, two-sided pricing, and the all-or-nothing transaction |
| [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) | Loot mode, and the sell-then-buy ownership primitive every transfer uses |
| [`UIActorMenuUpgrade.cpp`](UIActorMenuUpgrade.cpp.md) | Upgrade mode: the bench selection and separating an item from its stack |
| [`UIActorMenu_script.cpp`](UIActorMenu_script.cpp.md) | Repair and upgrade policy delegated to script, and the frozen script surface |
| [`UITradeWnd.cpp`](UITradeWnd.cpp.md) · [`UITradeWnd.h`](UITradeWnd.h.md) | The older standalone trade screen, superseded by the inventory's trade mode |

### Cells, grids and drag-and-drop

| Twin | Role |
|---|---|
| [`UICellItem.cpp`](UICellItem.cpp.md) · [`UICellItem.h`](UICellItem.h.md) | One item in a grid: stacking, splitting, how a press becomes a drag, and the floating drag widget |
| [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) | The three cell kinds — item, ammunition, weapon — and their three stacking rules |
| [`UICellItemFactory.cpp`](UICellItemFactory.cpp.md) · [`UICellItemFactory.h`](UICellItemFactory.h.md) | The single dispatch from an item to its cell kind |
| [`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) | The cell grid: placement, auto-grow, grouping, hover-resolved drop targets, the gesture callbacks |
| [`UIDragDropReferenceList.cpp`](UIDragDropReferenceList.cpp.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) | The quick slots: cells holding a section name rather than an item, and replace-on-drop |
| [`UIOutfitSlot.cpp`](UIOutfitSlot.cpp.md) · [`UIOutfitSlot.h`](UIOutfitSlot.h.md) | The outfit slot, which draws the worn suit behind its cell |

### Item description panels

| Twin | Role |
|---|---|
| [`UIItemInfo.cpp`](UIItemInfo.cpp.md) · [`UIItemInfo.h`](UIItemInfo.h.md) | The item description: name, icon, text, price, weight, and the sub-panel per item kind |
| [`UIWpnParams.cpp`](UIWpnParams.cpp.md) · [`UIWpnParams.h`](UIWpnParams.h.md) | A weapon's statistics, shown as bars against the class maximum |
| [`UIOutfitInfo.cpp`](UIOutfitInfo.cpp.md) · [`UIOutfitInfo.h`](UIOutfitInfo.h.md) | A suit's protection, per hit type, with a comparison against what is worn |
| [`UIBoosterInfo.cpp`](UIBoosterInfo.cpp.md) · [`UIBoosterInfo.h`](UIBoosterInfo.h.md) | What a consumable will do to you, normalised against the level's worst |
| [`ui_af_params.cpp`](ui_af_params.cpp.md) · [`ui_af_params.h`](ui_af_params.h.md) | An artefact's effects, shown the same way |

### Upgrades

| Twin | Role |
|---|---|
| [`UIInventoryUpgradeWnd.cpp`](UIInventoryUpgradeWnd.cpp.md) · [`UIInventoryUpgradeWnd.h`](UIInventoryUpgradeWnd.h.md) | The upgrade bench: the tree of available upgrades for the item on it |
| [`UIInventoryUpgradeWnd_add.cpp`](UIInventoryUpgradeWnd_add.cpp.md) | Building that tree from the item's upgrade data |
| [`UIInvUpgrade.cpp`](UIInvUpgrade.cpp.md) · [`UIInvUpgrade.h`](UIInvUpgrade.h.md) | One upgrade node, with its state and its connecting lines |
| [`UIInvUpgradeInfo.cpp`](UIInvUpgradeInfo.cpp.md) · [`UIInvUpgradeInfo.h`](UIInvUpgradeInfo.h.md) | The pop-up describing one upgrade: cost, effect, prerequisites |
| [`UIInvUpgradeProperty.cpp`](UIInvUpgradeProperty.cpp.md) · [`UIInvUpgradeProperty.h`](UIInvUpgradeProperty.h.md) | One property line of that description |

### Money, weight and standing

| Twin | Role |
|---|---|
| [`UITradeBar.cpp`](UITradeBar.cpp.md) · [`UITradeBar.h`](UITradeBar.h.md) | One side's offer total: price and weight |
| [`UIWeightBar.cpp`](UIWeightBar.cpp.md) · [`UIWeightBar.h`](UIWeightBar.h.md) | Carried weight against the carry limit, with the overload threshold |
| [`UIMoneyIndicator.cpp`](UIMoneyIndicator.cpp.md) · [`UIMoneyIndicator.h`](UIMoneyIndicator.h.md) | The money readout with its change animation |
| [`UICharacterInfo.cpp`](UICharacterInfo.cpp.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) | Portrait and particulars, read from the authoritative record so it works for someone not loaded |
| [`UIRankIndicator.cpp`](UIRankIndicator.cpp.md) · [`UIRankIndicator.h`](UIRankIndicator.h.md) | The rank as a row of pips |
| [`Restrictions.cpp`](Restrictions.cpp.md) · [`Restrictions.h`](Restrictions.h.md) | The multiplayer rank gates: what a rank may buy, and how many of each group it may carry |

### The in-game overlay

| Twin | Role |
|---|---|
| [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) | The overlay: what it owns, the ten-frame update throttle, and the draw-order exceptions |
| [`UIHudStatesWnd.cpp`](UIHudStatesWnd.cpp.md) · [`UIHudStatesWnd.h`](UIHudStatesWnd.h.md) | Health, armour, stamina, ammunition and the anomaly sensing with its peak-hold decay |
| [`UIMotionIcon.cpp`](UIMotionIcon.cpp.md) · [`UIMotionIcon.h`](UIMotionIcon.h.md) | The noise-and-light detectability icon |
| [`UIArtefactPanel.cpp`](UIArtefactPanel.cpp.md) · [`UIArtefactPanel.h`](UIArtefactPanel.h.md) | The belt artefact strip, drawn by reusing one primitive rather than owning widgets |
| [`ArtefactDetectorUI.cpp`](ArtefactDetectorUI.cpp.md) · [`ArtefactDetectorUI.h`](ArtefactDetectorUI.h.md) | The four detector displays, from a blinking lamp to a world-space map on the device's screen |
| [`UIMessagesWindow.cpp`](UIMessagesWindow.cpp.md) · [`UIMessagesWindow.h`](UIMessagesWindow.h.md) | The message area: the game log, the chat log and the chat entry, with two phase layouts |
| [`UIGameLog.cpp`](UIGameLog.cpp.md) · [`UIGameLog.h`](UIGameLog.h.md) | A scrolling log of timed lines that expire from the top |
| [`UIPdaKillMessage.cpp`](UIPdaKillMessage.cpp.md) · [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md) | One kill-feed line laid out from its four parts |
| [`KillMessageStruct.h`](KillMessageStruct.h.md) | The shape of that line: two coloured names and two icons, already localized |
| [`UICarPanel.cpp`](UICarPanel.cpp.md) · [`UICarPanel.h`](UICarPanel.h.md) | Empty: the removed vehicle dashboard |

### The personal terminal

| Twin | Role |
|---|---|
| [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) | The terminal: six pages behind a named tab control, with a script override and a deleted-page-is-fine policy |
| [`UIDiaryWnd.h`](UIDiaryWnd.h.md) | The diary page's declaration, with no implementation in this build |
| [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWnd.h`](UIMapWnd.h.md) | The map page: the global map, its level children, zoom and pan, the nine-button pad |
| [`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) | The map page's second half: hints, spot hit-testing and the level-change prompt |
| [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md) · [`UIMapWndActions.h`](UIMapWndActions.h.md) | Map navigation driven by a goal planner rather than a tween |
| [`UIMapWndActionsSpace.h`](UIMapWndActionsSpace.h.md) | That planner's world-state and operator identifiers |
| [`UIMap.cpp`](UIMap.cpp.md) · [`UIMap.h`](UIMap.h.md) | The three map kinds — global, level, minimap — and the per-frame rebuild of their spots |
| [`UIMapFilters.cpp`](UIMapFilters.cpp.md) · [`UIMapFilters.h`](UIMapFilters.h.md) | The four hardcoded spot categories and their check boxes |
| [`UIMapLegend.cpp`](UIMapLegend.cpp.md) · [`UIMapLegend.h`](UIMapLegend.h.md) | The legend panel: an icon key with no binding to the live spot list |
| [`UIMapInfo.cpp`](UIMapInfo.cpp.md) · [`UIMapInfo.h`](UIMapInfo.h.md) · [`UIMapInfo_script.cpp`](UIMapInfo_script.cpp.md) | The level description panel and its script surface |
| [`map_hint.cpp`](map_hint.cpp.md) · [`map_hint.h`](map_hint.h.md) | The hover hint a map spot raises, and its dwell rule |
| [`UITaskWnd.cpp`](UITaskWnd.cpp.md) · [`UITaskWnd.h`](UITaskWnd.h.md) | The task list, its filters, and the link from a task to its map spots |
| [`UISecondTaskWnd.cpp`](UISecondTaskWnd.cpp.md) · [`UISecondTaskWnd.h`](UISecondTaskWnd.h.md) | The secondary task list |
| [`UILogsWnd.cpp`](UILogsWnd.cpp.md) · [`UILogsWnd.h`](UILogsWnd.h.md) | The news log, built on a parallel work item because building its rows is expensive |
| [`UINewsItemWnd.cpp`](UINewsItemWnd.cpp.md) · [`UINewsItemWnd.h`](UINewsItemWnd.h.md) | One news entry |
| [`UIPdaMsgListItem.cpp`](UIPdaMsgListItem.cpp.md) · [`UIPdaMsgListItem.h`](UIPdaMsgListItem.h.md) | One message row, in several authored shapes |
| [`UIActorInfo.cpp`](UIActorInfo.cpp.md) · [`UIActorInfo.h`](UIActorInfo.h.md) | The statistics page: a master list of categories and a detail list, with reputation as a non-statistic |
| [`UIAchievements.cpp`](UIAchievements.cpp.md) · [`UIAchievements.h`](UIAchievements.h.md) | An achievement row that polls a script predicate and joins or leaves its list |
| [`UIRankingWnd.cpp`](UIRankingWnd.cpp.md) · [`UIRankingWnd.h`](UIRankingWnd.h.md) | The ranking page |
| [`UIRankingsCoC.cpp`](UIRankingsCoC.cpp.md) · [`UIRankingsCoC.h`](UIRankingsCoC.h.md) | A later game's replacement ranking page |
| [`UIRankFaction.cpp`](UIRankFaction.cpp.md) · [`UIRankFaction.h`](UIRankFaction.h.md) | One faction's standing row on the ranking page |
| [`UIFactionWarWnd.cpp`](UIFactionWarWnd.cpp.md) · [`UIFactionWarWnd.h`](UIFactionWarWnd.h.md) | The faction war page, refreshed on a time interval scaled by the time factor |
| [`UIWarState.cpp`](UIWarState.cpp.md) · [`UIWarState.h`](UIWarState.h.md) | One war-state indicator on that page |
| [`FactionState.cpp`](FactionState.cpp.md) · [`FactionState.h`](FactionState.h.md) · [`FactionState_inline.h`](FactionState_inline.h.md) | One faction's row: the engine supplies identity and standing, a script supplies the rest |
| [`FractionState.cpp`](FractionState.cpp.md) · [`FractionState.h`](FractionState.h.md) · [`FractionState_inline.h`](FractionState_inline.h.md) | The earlier spelling of the same record, kept because its script name is frozen |

### The dialogue screen

| Twin | Role |
|---|---|
| [`UITalkWnd.cpp`](UITalkWnd.cpp.md) · [`UITalkWnd.h`](UITalkWnd.h.md) | The conversation: turn-taking, the phrase graph step, voice, camera easing, and the handoff to trade |
| [`UITalkDialogWnd.cpp`](UITalkDialogWnd.cpp.md) · [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md) | Its view: two scrolling lists, two character panels, three buttons, one clicked-phrase mailbox |
| [`UISpeechMenu.cpp`](UISpeechMenu.cpp.md) · [`UISpeechMenu.h`](UISpeechMenu.h.md) | The cursorless radial order menu for squad commands |

### The server browser and joining

| Twin | Role |
|---|---|
| [`ServerList.cpp`](ServerList.cpp.md) · [`ServerList.h`](ServerList.h.md) | The browser: the row pool, the deferred refresh, the filters and sorts, and what a master server must answer |
| [`ServerList_GameSpy_func.cpp`](ServerList_GameSpy_func.cpp.md) | Emptied: the vendor-service callbacks, deleted when the service died |
| [`UIListItemServer.cpp`](UIListItemServer.cpp.md) · [`UIListItemServer.h`](UIListItemServer.h.md) | One browser row, and composing the join console command |
| [`UIServerInfo.cpp`](UIServerInfo.cpp.md) · [`UIServerInfo.h`](UIServerInfo.h.md) | The in-game server information panel |
| [`UICDkey.cpp`](UICDkey.cpp.md) · [`UICDkey.h`](UICDkey.h.md) | The masked product-key field, the player-name field, and the per-machine settings store |
| [`UIOptConCom.cpp`](UIOptConCom.cpp.md) · [`UIOptConCom.h`](UIOptConCom.h.md) | Settings controls that write console commands instead of console variables |
| [`UIMapList.cpp`](UIMapList.cpp.md) · [`UIMapList.h`](UIMapList.h.md) | The level and weather catalogue per game type, and the server's map rotation editor |

### Multiplayer votes and administration

| Twin | Role |
|---|---|
| [`UIChangeMap.cpp`](UIChangeMap.cpp.md) · [`UIChangeMap.h`](UIChangeMap.h.md) | Vote to change the level, with a preview found by naming convention |
| [`ChangeWeatherDialog.cpp`](ChangeWeatherDialog.cpp.md) · [`ChangeWeatherDialog.hpp`](ChangeWeatherDialog.hpp.md) | A numbered button list with digit shortcuts, and the weather and game-type votes on it |
| [`UIVote.cpp`](UIVote.cpp.md) · [`UIVote.h`](UIVote.h.md) | The vote prompt and its two answers |
| [`UITextVote.cpp`](UITextVote.cpp.md) · [`UITextVote.h`](UITextVote.h.md) | A free-text vote |
| [`UIVoteStatusWnd.cpp`](UIVoteStatusWnd.cpp.md) · [`UIVoteStatusWnd.h`](UIVoteStatusWnd.h.md) | The running tally while a vote is open |
| [`UIVotingCategory.cpp`](UIVotingCategory.cpp.md) · [`UIVotingCategory.h`](UIVotingCategory.h.md) | The menu of what may be voted on |
| [`UIKickPlayer.cpp`](UIKickPlayer.cpp.md) · [`UIKickPlayer.h`](UIKickPlayer.h.md) | Vote to remove a player |
| [`UIMPAdminMenu.cpp`](UIMPAdminMenu.cpp.md) · [`UIMPAdminMenu.h`](UIMPAdminMenu.h.md) | The administrator menu's tabbed shell |
| [`UIMPPlayersAdm.cpp`](UIMPPlayersAdm.cpp.md) · [`UIMPPlayersAdm.h`](UIMPPlayersAdm.h.md) | Its player tab |
| [`UIMPServerAdm.cpp`](UIMPServerAdm.cpp.md) · [`UIMPServerAdm.h`](UIMPServerAdm.h.md) | Its server-settings tab |
| [`UIMPChangeMapAdm.cpp`](UIMPChangeMapAdm.cpp.md) · [`UIMPChangeMapAdm.h`](UIMPChangeMapAdm.h.md) | Its level-change tab, which acts directly instead of voting |
| [`UIDemoPlayControl.cpp`](UIDemoPlayControl.cpp.md) · [`UIDemoPlayControl.h`](UIDemoPlayControl.h.md) | Demo playback transport, one of the few screens that works while paused |

### The multiplayer buy menu

| Twin | Role |
|---|---|
| [`UIBuyWndBase.h`](UIBuyWndBase.h.md) | The interface a buy menu must satisfy, and the seven preset loadout slots |
| [`UIBuyWndShared.cpp`](UIBuyWndShared.cpp.md) · [`UIBuyWndShared.h`](UIBuyWndShared.h.md) | The item catalogue: five rank prices and a tab per item, in a name-sorted table whose index order is an identifier |
| [`UIMpTradeWnd.cpp`](UIMpTradeWnd.cpp.md) · [`UIMpTradeWnd.h`](UIMpTradeWnd.h.md) | The buy menu itself |
| [`UIMpTradeWnd_init.cpp`](UIMpTradeWnd_init.cpp.md) | Its construction from layout and catalogue |
| [`UIMpTradeWnd_items.cpp`](UIMpTradeWnd_items.cpp.md) | Filling its lists and moving items between them |
| [`UIMpTradeWnd_trade.cpp`](UIMpTradeWnd_trade.cpp.md) | Pricing, affordability and the commit |
| [`UIMpTradeWnd_wpn.cpp`](UIMpTradeWnd_wpn.cpp.md) | Weapons and their attachments, which are bought together |
| [`UIMpTradeWnd_misc.cpp`](UIMpTradeWnd_misc.cpp.md) | Presets, the money readout and the remainder |
| [`UIMpItemsStoreWnd.cpp`](UIMpItemsStoreWnd.cpp.md) · [`UIMpItemsStoreWnd.h`](UIMpItemsStoreWnd.h.md) | The later store screen built on the same catalogue |
| [`UIBuyWeaponTab.cpp`](UIBuyWeaponTab.cpp.md) · [`UIBuyWeaponTab.h`](UIBuyWeaponTab.h.md) | Its tab strip, which fires the change callback even for the tab already selected |
| [`UITabButtonMP.cpp`](UITabButtonMP.cpp.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) | A tab button carrying a count badge |

### Multiplayer round screens

| Twin | Role |
|---|---|
| [`UISpawnWnd.cpp`](UISpawnWnd.cpp.md) · [`UISpawnWnd.h`](UISpawnWnd.h.md) | The spawn point and team choice |
| [`UISkinSelector.cpp`](UISkinSelector.cpp.md) · [`UISkinSelector.h`](UISkinSelector.h.md) | The character model choice, filtered by the extra-content gate |
| [`UIChatWnd.cpp`](UIChatWnd.cpp.md) · [`UIChatWnd.h`](UIChatWnd.h.md) | The chat entry line, cursorless, with two phase layouts |
| [`UIStats.cpp`](UIStats.cpp.md) · [`UIStats.h`](UIStats.h.md) | The scoreboard's frame and columns |
| [`UIStatsPlayerList.cpp`](UIStatsPlayerList.cpp.md) · [`UIStatsPlayerList.h`](UIStatsPlayerList.h.md) | Its player rows, grouped by team |
| [`UIStatsPlayerInfo.cpp`](UIStatsPlayerInfo.cpp.md) · [`UIStatsPlayerInfo.h`](UIStatsPlayerInfo.h.md) | One player's row |
| [`UIStatsIcon.cpp`](UIStatsIcon.cpp.md) · [`UIStatsIcon.h`](UIStatsIcon.h.md) | The per-player status icons |
| [`UIFrags.cpp`](UIFrags.cpp.md) · [`UIFrags.h`](UIFrags.h.md) · [`UIFrags2.cpp`](UIFrags2.cpp.md) · [`UIFrags2.h`](UIFrags2.h.md) | The two score readouts, one per game-type family |
| [`TeamInfo.cpp`](TeamInfo.cpp.md) · [`TeamInfo.h`](TeamInfo.h.md) | The two teams' names and colours, resolved once and cached process-wide |

## Not given twins

The directory contains only source; the build description lives one level up. What it records
and this chapter relies on: the game module links against the toolkit, the engine, the core,
the script engine and the vendor matchmaking wrapper, and the last of those is unconditional —
there is no build option to leave it out.

## What could not be recovered

Collected from across the chapter, for the recipe's honesty section.

- **The 155 alpha on team colours** and the matching 255 on the team colour markup: two
  different opacities for the same colour in two contexts, with no recorded reason beyond
  "tints read over the world, text does not".
- **Team 3 is displayed as team 2** throughout. Which game type has a third team, and whether
  it was ever meant to have its own name, is not discoverable from this directory.
- **The quantisation denominators** — 55 health segments, 31 protection segments, 15 for a
  list's condition bar, 13 for a cell's, 35 for the overlay's health bar — are properties of
  textures that are not in this repository. They cannot be re-derived; they must be copied.
- **The consumable-use cut at 8 remaining uses**, above which the bar is continuous and below
  which it is eighths offset by half an eighth. The 8 matches a background-less bar variant,
  but nothing states that.
- **The 30-unit growth added to an achievement row's description**: the space above it, frozen
  against a row layout that is not described anywhere.
- **The four custom use actions** in the context menu. Four, with four distinct action tags
  and no reason for the number.
- **The hardcoded section names** in the use-action labelling (`vodka`, `bread`, and three
  others) and the surge-survival drug's section name in the booster panel. Both are flagged in
  the source as things to remove; both are load-bearing against shipped data.
- **The crossed axes in the artefact panel's icon sizing** — width taken from the rectangle's
  height. Almost certainly a slip, invisible because the shipped icons are square, and
  therefore impossible to settle from the data.
- **The colour-channel swap** in the UI's colour-animation playback. Whether the world
  renderer or the UI emitter is the one with the unconventional order is not decidable here.
- **The product-key format** — four groups of four, sixteen characters, stored hyphenated —
  was defined by a service that no longer exists. Nothing validates it locally beyond length.
- **The `"foo"` and `"total"` sentinel identifiers** in the statistics page's master list. Both
  are frozen by a shipped document and neither is explained.
- **`SServerFilters`' unused field.** It is declared, never read by the engine, and carries a
  comment saying one game's scripts need it — which is the only evidence of what it was for.
- **The 3-metre interaction distance** that closes trade and upgrade. It matches other
  distances in the game but is not shared with them.
- **Why loot mode was exempted from that distance check** while trade and upgrade were not.
  The exemption is commented as deliberate and unexplained.
