# src/xrGame/ui/UIChangeMap.cpp

> Pick a level, see its picture, and start a vote to switch to it.

**Needs** — [`UIChangeMap.h`](UIChangeMap.h.md) · [`UIMapList.h`](UIMapList.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../../xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md)
**Used by** — [`UIChangeMap.h`](UIChangeMap.h.md)
**Tier floor** — T3: list plus a console command

## Purpose

One of a small family of multiplayer vote dialogs. What is worth recording is not the widgets
but the **shape of the whole family**: a dialog collects a choice, formats it into a console
command string, executes that string, and closes. The vote itself, the network round trip and
the result are none of the dialog's business.

A rebuild replaces the console-command channel with whatever it likes, but must keep the
separation: these dialogs are input devices for the console, not participants in the vote.

## State

The selected index, and the two widgets that depend on it — a preview picture and a version
label. The level list itself is not owned: it comes from a process-wide map-list helper keyed
by the current game type.

## `InitChangeMap`

**Contract** — Build every widget by name from the document. Three are decorative and the
result is deliberately discarded; four are kept: the preview picture, the version label, the
list and the two buttons. Two frame windows are optional; everything else is required. Then
fill the list.

## `FillUpList`

**Contract** — One row per level available for the current game type, in the helper's order,
showing the level's **localized** name. Every row is enabled.

**Invariants** — The row's *index* is the identity; the displayed text is localized and the
command below uses the unlocalized name from the same index. Sorting or filtering the rows
without re-deriving that mapping breaks the command.

## `OnItemSelect`

**Contract** — Show the selected level's preview and version.

```text
FUNCTION on_item_select()
  idx = the list's selection; RETURN IF none
  entry = the map list for this game type, at idx

  version_label = "[" + (entry.version or "unknown") + "]"

  # The preview is found by convention, not by configuration: a texture
  # named after the level under a fixed prefix.
  path = "intro/intro_map_pic_" + entry.name
  remember the picture's current texture rectangle
  IF the texture file exists THEN show it ELSE show the noise placeholder
  restore the remembered texture rectangle
```

**Invariants** — The preview is a **naming convention**, so a new level gets a preview by
being shipped with a correctly named texture and needs no data entry. The noise placeholder is
the graceful failure.

Saving and restoring the texture rectangle around the change is necessary because binding a
texture resets it to the whole image, and the layout authored a sub-rectangle.

## `OnBtnOk`

**Contract** — Format `changemap <level> <version>` as a vote-start console command and
execute it, then close. The index is bounds-checked against the list, because the list can be
refilled while the dialog is open.

## `OnBtnCancel` / `OnKeyboardAction`

**Contract** — Close. The quit binding is the same as cancel.

## `SendMessage`

**Contract** — Routes the list's selection notification and the two buttons' click
notifications to the three handlers, by widget identity. Note it does **not** delegate
unhandled notifications to the base, so this dialog swallows every notification its children
send.
