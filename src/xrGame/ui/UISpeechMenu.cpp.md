# src/xrGame/ui/UISpeechMenu.cpp

> The multiplayer quick-chat menu: numbered canned phrases read from configuration, dismissed by the
> digit key that picks one.

**Needs** — [`UISpeechMenu.h`](UISpeechMenu.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../UIGameCustom.h`](../UIGameCustom.h.md) · [`../Level.h`](../Level.h.md) · [`../game_cl_mp.h`](../game_cl_mp.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UISpeechMenu.h`](UISpeechMenu.h.md)
**Tier floor** — T3: a list built from configuration and a key-to-index map

## Purpose

A menu the player opens mid-round to send one of a handful of canned phrases to their team. It is
deliberately the least intrusive screen in the chapter: no cursor, no movement lock, no background,
and it closes the instant a digit is pressed. The phrase set is configuration, not code, so a mod
can change it without touching the engine.

## State

```text
RECORD SpeechMenu extends ModalDialog
  list       : ScrollContainer   # one text row per phrase, in configuration order
  text_colour: int (packed RGBA)
  font       : Font
```

Invariants:

- Row *i* of the list corresponds to phrase index *i*, and the digit key `i+1` selects it. The row's
  visible number is `i+1` while the index sent to the game is `i` — the off-by-one is the whole
  contract between this screen and the game state.

## `CUISpeechMenu`

**Contract** — Builds from the shared in-game layout document `maingame.xml` under the element
`speech_menu`, which supplies both the window geometry and the scroll container. The list is then
**re-parked at the window's origin**, overriding whatever position the document gave it, so the
list always fills the menu. Reads the row font and colour from `speech_menu:text`. Then populates.

## `InitList`

**Contract** — Reads phrases from the named configuration section by probing consecutive keys
`phrase_0`, `phrase_1`, … until one is missing. Each value's first comma-separated field is a
localization identifier; the row's text is the ordinal, a dot, and the translated string. Fails hard
if the section does not exist.

```text
FUNCTION InitList(section)
  REQUIRE configuration has section
  FOR i FROM 0 WHILE section has key "phrase_" + i
    id  <- first comma-separated field of section["phrase_" + i]
    row <- text row: decimal(i + 1) + ". " + localized(id)
    row.font   <- font
    row.colour <- text_colour
    list.insert(row)
```

**Notes** — Probing for consecutive keys rather than reading a count means a gap in the numbering
silently truncates the menu. That is the same convention the rest of the configuration uses and a
rebuild should keep it, because shipped data relies on the termination rule.

Only the *first* field of each value is used; the rest of the line carries data for other systems
(the sound to play, for one) that this screen does not read.

## `OnKeyboardAction`

**Contract** — Any key in the digit range closes the menu and reports the chosen index, zero-based,
to the multiplayer game state — which is what actually sends the phrase. Anything else falls through
to normal dialog handling, so the menu's own close key still works.

```text
FUNCTION OnKeyboardAction(scancode, action)
  IF scancode IS NOT in the digit-key range THEN RETURN base.OnKeyboardAction(scancode, action)
  hide()
  game.on_phrase_chosen(self, scancode - first digit key)
  RETURN handled
```

**Notes** — The range test admits every digit key including zero, and the index it computes for the
zero key is one past the nine keys before it — so a tenth phrase is reachable, and an eleventh is
not. The handler does not check the index against the number of phrases; a digit beyond the end
reaches the game state, which is where the range is actually enforced.

The reporting path runs through the multiplayer game state and out to a service described at
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts).
