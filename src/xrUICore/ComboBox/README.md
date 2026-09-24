# src/xrUICore/ComboBox — the drop-down

> A closed text line that expands into a list drawn above everything else, holding focus and
> the pointer for the duration, and bound to a console variable's token set.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The one control in the chapter that has to break the tree's drawing rules. While open, its
list must appear over every sibling regardless of where it sits in the tree, and every pointer
event in the screen must reach it so that a click outside closes it.

It is also a settings control: its item set is a console variable's token set, and its
selection is that variable's value.

## The load-bearing ideas

**Opening changes three things at once.** The widget grows to hold its list, locks navigation
focus to itself, and captures the pointer. Closing undoes all three. Any rebuild that forgets
one of the three gets a drop-down that is either unreachable by gamepad, unclosable by a click
elsewhere, or clipped by its parent.

**The open list is drawn after the rest of the interface**, not in tree order — the same
deferral the cursor and the tooltips use, and for the same reason.

**A selection is a token, not an index.** The item set comes from the console variable's
declared tokens, so the stored value is a name and the displayed text is that name's
localization. Two items whose localized names collide become indistinguishable, and a missing
translation matches nothing.

**It dresses itself by probing.** Two generations of the game data use different texture names
for the drop-down's parts, and the widget tries both. A rebuild targeting only one game's data
can drop the probe; one that must load all three cannot.

## The twins

| Twin | Role |
|---|---|
| [`UIComboBox.cpp`](UIComboBox.cpp.md) | Probing two texture generations, growing while open, locking focus and capturing the pointer, the deferred draw, and the console token mapping |
| [`UIComboBox.h`](UIComboBox.h.md) | Its declaration |
