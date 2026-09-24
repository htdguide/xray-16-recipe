# src/xrGame/ui/ChangeWeatherDialog.cpp

> A dialog that is a numbered list of buttons, and the two votes built from it.

**Needs** — [`ChangeWeatherDialog.hpp`](ChangeWeatherDialog.hpp.md) · [`UIMapList.h`](UIMapList.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`../../xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md)
**Used by** — [`ChangeWeatherDialog.hpp`](ChangeWeatherDialog.hpp.md)
**Tier floor** — T3: button routing plus a console command

## Purpose

Two vote dialogs share one shape — a header, a background, a cancel, and *n* numbered buttons
— and the shape is worth extracting because the interesting behaviour lives in it: the buttons
are reachable by number key, and the label text is generated rather than authored.

Like [`UIChangeMap.cpp`](UIChangeMap.cpp.md), both dialogs end in a console command; the vote
itself is not theirs.

## `ButtonListDialog`

### State

```text
RECORD ButtonListDialog
  header, background, cancel : widgets
  buttons : list<(button, label)>     # created by Initialize, count decided by the subclass
```

### `Initialize`

**Contract** — Create *n* (button, label) pairs and attach them. Geometry comes later, per
button, from the layout document; this only decides how many exist.

### `OnKeyboardAction`

**Contract** — After the widget tree has had the key:

```text
FUNCTION on_key(key, action)
  IF action is not a press THEN RETURN not handled
  IF the key is the quit binding THEN cancel; RETURN handled

  # Buttons 1..n are reachable by the digit keys 1..9. Only when there
  # are nine or fewer: with ten or more the mapping would be ambiguous
  # and no digit selects anything.
  IF button count <= 9 AND the key is a digit within range
    click that button
    RETURN handled
```

**Invariants** — The nine-button limit is the decision, not the digit range. A list of ten
weather sets silently loses all keyboard shortcuts rather than giving nine of them a shortcut
and leaving one without — which the author judged more confusing.

### `SendMessage`

**Contract** — On a click, cancel if it was the cancel button, otherwise find the button by
identity and report its index. Note the cancel arm does not return, so a cancel also runs the
search — harmless, since cancel is not in the list.

## `ChangeWeatherDialog::InitChangeWeather`

**Contract** — Build from the document, then **size the button list from the data**: one
button per weather set the map-list helper offers. Each button's geometry comes from an
element named by position — `btn_1`, `btn_2`, … — and so does each label's.

The label's *text*, however, is overwritten:

```text
FOR i IN 0 .. weather count - 1
  configure button i   from element "btn_" + (i+1)
  configure label i    from element "txt_" + (i+1)

  # The document's own label text is DISCARDED. The XML reader preserves
  # document order while the configuration reader does not, so the two
  # orders can disagree; generating the label from the weather list makes
  # the list the single source of truth.
  label[i].text = (i+1) + ". " + localize(weather[i].name)
  remember weather[i].name and weather[i].start_time
```

**Invariants** — This is the file's real content. Two readers over the same authored data
disagree about order, and the fix is to stop trusting one of them for anything but geometry.
A rebuild whose configuration reader preserves order does not need the override — but does
need to *decide*, because the bug is silent: the buttons look right and vote for the wrong
weather.

The number prefix in the label is what makes the keyboard shortcut discoverable.

## `ChangeWeatherDialog::OnButtonClick`

**Contract** — Format `changeweather <name> <start time>` as a vote-start console command,
execute it, close.

## `ChangeGameTypeDialog::InitChangeGameType`

**Contract** — The same shape with **four** buttons, a count hardcoded because the engine has
no table of game types to ask; the source flags this. Each button's identifier is read from an
attribute on its *label* element, so the game types offered are decided entirely by the layout
document.

**Notes** — Reading the identifier from the label element rather than the button element is
arbitrary and a rebuild may put it anywhere; what matters is that the identifier travels with
the authored entry, since the code has no list of its own.

## `ChangeGameTypeDialog::OnButtonClick`

**Contract** — Format `changegametype <identifier>` as a vote-start console command, execute
it, close.
