# src/xrGame/ui/UISkinSelector.cpp

> The multiplayer skin picker: a fixed-width window onto a longer list of skins, paged left and
> right, selectable by picture, by digit key or at random.

**Needs** — [`UISkinSelector.h`](UISkinSelector.h.md) · [`UIStatix.h`](UIStatix.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`../UIGameCustom.h`](../UIGameCustom.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_deathmatch.h`](../game_cl_deathmatch.h.md) · [`xrUICore/Static/UIAnimatedStatic.h`](../../xrUICore/Static/UIAnimatedStatic.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UISkinSelector.h`](UISkinSelector.h.md)
**Tier floor** — T3: list paging and selection over a configuration-supplied name list

## Purpose

A multiplayer round needs the player to pick a character model before spawning. The number of skins
is a configuration value and the number of picture slots is a layout value, and they do not agree —
so the screen is a **window onto the list**: the layout supplies however many picture slots it
contains, and the screen scrolls the list underneath them.

That one decision accounts for nearly everything in the file: the slot count is discovered from the
layout, the window offset is the only thing the paging buttons change, the digit shortcuts address
the *list*, not the slots, and the arrow buttons enable themselves from whether the window can move.

## State

```text
RECORD SkinSelector extends ModalDialog
  section        : text          # configuration section naming the skins
  skins          : list<text>    # every skin name, in configuration order
  enabled        : list<int>     # indices of skins the player may choose
  slots          : list<Picture> # the layout's picture slots; count discovered, >= 1
  window_start   : int           # index of the skin shown in slot 0
  active         : int           # chosen skin index, or -1 meaning "not chosen"
  team           : int (16-bit)
  page_buttons   : [left, right]                 # optional
  page_animations: [left, right]                 # optional; played on focus
  shader         : optional<text>                # alternate material for skin pictures
  back, spectator, auto_select : Button
```

Invariants:

- `0 <= window_start` and `window_start + slots.count <= skins.count` — enforced by the two paging
  operations, which refuse to move past either end.
- `active` is either -1 or a valid index into `skins`; -1 is not "none", it means **choose at
  random from the enabled set on accept**.
- Slot *i* always shows skin `window_start + i`. There is no other mapping.
- `enabled` starts as every index; the multiplayer game state narrows it.

## `Init`

**Contract** — Builds the screen from the fixed layout document `skin_selector.xml`. Discovers the
number of picture slots by probing for numbered elements `skin_selector:image_0`,
`…image_1`, … until one is missing — with the rule that **element zero must exist**, so a layout
with no slots is a hard failure rather than an empty screen. Reads the optional alternate material
name for skin pictures, then loads the skin list and paints the slots.

The element names are frozen: `skin_selector` and beneath it `background`, `caption`,
`image_frames` (holding `a_static_1`, `a_static_2`, `btn_left`, `btn_right`), the numbered
`image_N`, and `btn_autoselect`, `btn_spectator`, `btn_back`, `skin_shader`.

**Notes** — The paging buttons live inside the frame element and have their message target
redirected to the screen, because their notifications must be distinguishable from the picture
slots' notifications — both arrive as the same message id and are told apart only by sender.

## `InitSkins`

**Contract** — Reads a comma-separated skin list out of the named configuration section and fails
hard if the section, the key, or any entries are missing. Every index starts enabled.

**Notes** — Failing hard rather than showing an empty picker is deliberate: an empty skin list means
the game's multiplayer configuration is broken, and a player left staring at a blank screen with no
way to spawn is worse than a stop.

## `UpdateSkins`

**Contract** — Repaints every slot from the current window. Runs on every frame (via `Update`) as
well as on every change, so nothing has to remember to call it.

```text
FUNCTION UpdateSkins()
  FOR i IN 0 .. slots.count - 1
    index <- window_start + i
    slots[i].bind_texture(skins[index], with the alternate material if one was configured)
    slots[i].selected <- (index == active)
    IF index < 10 THEN
      slots[i].text <- decimal((index + 1) MOD 10) + " "   # the digit key that picks it
    ELSE
      slots[i].text <- ""
    slots[i].enabled <- index IS IN enabled
  page_buttons.left.enabled  <- window_start > 0
  page_buttons.right.enabled <- window_start + slots.count < skins.count
```

**Notes** — The label arithmetic is the keyboard layout, not the list: the tenth skin is labelled
`0` because it is picked with the zero key, which sits to the right of `9`. Only the first ten skins
get a shortcut and a label; the rest are pointer-only. The trailing space in the label is authored
padding between the number and the picture edge.

## `SetCurSkin`

**Contract** — Sets the chosen index and, if that index is outside the current window, scrolls the
window to bring it into view — clamped so the window never runs off the end of the list. Rejects an
index outside the list. Called by the multiplayer game state to restore a previous choice.

```text
FUNCTION SetCurSkin(index)
  REQUIRE -1 <= index <= skins.count
  active <- index
  IF index != -1 AND index IS outside the window THEN
    IF index > skins.count - slots.count THEN window_start <- skins.count - slots.count
    ELSE                                      window_start <- index
  UpdateSkins()
```

## Choosing a skin

**Contract** — Three routes, all ending in the same accept:

- clicking a slot sets `active` to that slot's skin and accepts;
- pressing a digit key sets `active` to that *list* index and accepts, but **only if that index is
  in the enabled set** — a disabled skin's key does nothing rather than failing;
- auto-select sets `active` to -1 and accepts.

Accept hides the screen, resolves `active == -1` into a **uniformly random choice from the enabled
set**, and tells the multiplayer game state. Cancel hides and tells the game state it was cancelled.

```text
FUNCTION accept()
  hide()
  IF active == -1 THEN active <- enabled[random index into enabled]
  game.on_skin_chosen()
```

**Notes** — The random resolution happens after hiding, so the player never sees the picker jump to
the randomly chosen skin. The digit range is clamped to nine slots even when ten skins are labelled,
because the keys checked are the contiguous run starting at `1`; the zero key is labelled but not
matched. That is a defect in the shipped code — the label promises a shortcut the handler does not
implement — and a rebuild that wires the zero key up would be a strict improvement, not a deviation
anything depends on.

## `OnKeyboardAction`

**Contract** — Four bound actions and one raw key range.

- The **scores** action, held, hides every child and passes the press through to the multiplayer
  game state, which shows the scoreboard; releasing it restores the children and the cursor. The
  screen stays up underneath. This is how the scoreboard is reachable from a screen that otherwise
  swallows the keyboard.
- Digit keys pick a skin as above.
- Quit cancels; jump auto-selects and accepts; enter accepts the current choice; left and right page
  the window.

**Notes** — The scores handling is the reason this method deals with both press and release; every
other action ignores releases.

## `Update`

**Contract** — Repaints the slots and defers to the base. Repainting unconditionally every frame is
what keeps the enabled set, which the game state can change at any time, reflected in the slots
without any change notification.
