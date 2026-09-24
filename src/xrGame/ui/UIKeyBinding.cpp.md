# src/xrGame/ui/UIKeyBinding.cpp

> The controls page: builds a scrolling table of grouped actions from a shipped description document, gives each row one or two binding cells, and re-reads every cell whenever the binding table changes behind its back.

**Needs** — [`UIKeyBinding.h`](UIKeyBinding.h.md) · [`UIEditKeyBind.h`](UIEditKeyBind.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/XML/xrUIXmlParser.h`](../../xrUICore/XML/xrUIXmlParser.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIKeyBinding.h`](UIKeyBinding.h.md)
**Tier floor** — T3.

## Purpose

**Two documents, not one.** The layout document says what a group heading and a table row
*look* like; a separate description document says which actions exist, how they are grouped,
and in what order. The page reads the second for structure and the first for appearance, and
instantiates one row per command.

That split is the load-bearing decision. It means the controls page's *content* is game data
— a mod can add an action row without touching a layout — while its appearance stays under
the style mechanism with every other screen.

## State

```text
RECORD KeyBindingPage EXTENDS Window, KeyMapChangeWatcher
  gamepad_mode : bool              # read from the layout, not chosen in code
  headers      : list<FrameLine>   # three: action, primary, secondary
  frame        : FrameWindow
  rows         : ScrollView        # group headings and command rows, interleaved
```

**Invariants**

- In gamepad mode there is one binding column and the third header is never dressed; in
  keyboard mode there are two. The page is otherwise identical, which is why one class
  serves both and the mode is a layout attribute.
- Each binding cell is positioned and sized from its **header's** position and width, minus
  three units. Columns therefore line up with their headings by construction rather than by
  the layout repeating the geometry, and moving a header in the layout moves its column.

## Building the table

**Contract** —

```text
FUNCTION fill(layout, path)
  description := load "ui_keybinding.xml" or "ui_keybinding_gamepad.xml" by mode
  FOR EACH group IN description
    heading := a Label dressed from layout path + ":scroll_view:item_group"
    heading.text := localize(group.name)
    append heading to the scroll view
    FOR EACH command IN group
      row := a Label dressed from layout path + ":scroll_view:item_key"
      row.text := localize(command.id)
      append row to the scroll view
      action := command.exe                      # the canonical action name
      attach a binding cell to the row for the primary slot, sized from header 2
      IF NOT gamepad_mode
        attach a second binding cell for the secondary slot, sized from header 3
```

**Invariants** — a command carries two names: an `id` that is a localization identifier for
the player-facing label, and an `exe` that is the engine's canonical action name. They are
different strings and conflating them breaks either the display or the binding. The settings
group the cells are assigned to also differs by mode, so a broadcast from a keyboard cell
never reaches a gamepad cell — the conflict model is scoped per input kind.

**Notes** — the two element paths under `scroll_view` are the frozen contract with the
shipped layout: `item_group` for a heading, `item_key` for a row. The description document's
node and attribute names — `group`/`name`, `command`/`id`/`exe` — are likewise frozen,
because mods ship their own copies.

The binding cells are attached to the *row label*, not to the scroll view, so a row plus its
one or two cells is one scrollable unit of uniform height.

## Reacting to bindings changed elsewhere

**Contract** — the page registers as a watcher on the input layer's binding table. When the
table changes for any reason — a console command, a configuration reload, a default restore —
every binding cell on the page is told to re-read its slot.

**Notes** — this is the reason the page is a watcher rather than reading once at open. The
"restore defaults" button and the console both rewrite the whole table without going through
any cell, and a page that did not re-read would show stale keys and then commit them back.
Re-reading is safe because a cell's displayed value is derived, never authoritative.

## Development check

**Contract** — in development builds the page walks the engine's complete action list and
appends a visible row for every action the description document does not mention, under a
shouted heading.

**Notes** — the check is *in the screen*, not in a test, and its output is rows the developer
sees while using the page. That is the practical consequence of the two-document split: an
action added in code with no description entry would otherwise be simply unbindable and
invisible. A rebuild is free to move this to a build-time validation, which is strictly
better; what must survive is that the two documents are checked against each other at all.
