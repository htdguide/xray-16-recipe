# src/xrUICore/SpinBox/UISpinText.cpp

> A spin box over an ordered list of named choices, showing the localized label while storing and saving the untranslated name.

**Needs** — [`UISpinText.h`](UISpinText.h.md) · [`UICustomSpin.h`](UICustomSpin.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`xrCore/xr_token.h`](../../xrCore/xr_token.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UISpinText.h`](UISpinText.h.md)
**Tier floor** — T3: a cursor over a list plus a localization lookup.

## Purpose

The options-screen control for a setting whose values are named rather than numeric — a
renderer, a language, a quality preset. The value set is not authored into the widget: it
comes from the setting's own *token list*, the engine-wide mechanism that pairs a stored name
with a numeric identifier. The spin box turns that list into a linear sequence the player
walks with two arrows.

The decision worth extracting is that each entry carries **both** spellings of itself: the
original name, which is what the setting file stores and what the engine compares, and the
translated label, which is what the player reads. Losing either breaks something — losing the
original writes a localized string into a settings file that a differently-localized run
cannot read back.

## State

```text
RECORD SpinText EXTENDS CustomSpin
  entries  : list<Choice>
  current  : int            # index into entries; -1 means "no entries yet"
  backup   : int

RECORD Choice
  original   : text    # the stored, comparable spelling
  translated : text    # the displayed spelling, via the string table
  id         : int     # the token's numeric identifier
```

**Invariants** — `current` is -1 only before the first entry is added; the first added entry
sets it to zero and paints itself. The identifier is recorded and never used by this widget —
see the note below.

## `AddItem_`

**Contract** — appends a choice, translating its name through the string table at insertion
time, and selects it if it is the first. Translating once on insert rather than on every draw
is the decision; it means a language change after a screen is built does not restyle it, which
is acceptable because changing language rebuilds the screen.

## `SetCurrentOptValue`

**Contract** — the read half of the options protocol, and the only place the choice list is
populated.

```text
FUNCTION set_current_opt_value()
  FOR EACH token IN the setting's token list
    add_item(token.name, token.id)
  stored <- the setting's current value, as text
  FOR EACH entry, index IN entries
    IF entry.original == stored THEN current <- index ; BREAK
  display(entries[current])
```

**Notes** — a stored value that matches nothing leaves the cursor wherever it was — at zero
for a fresh box — so an unrecognized setting silently becomes the first choice rather than
failing. That is the right behaviour for a settings file written by a newer build, and it is
worth keeping deliberately.

Populating from the token list inside the *read* step means the list is rebuilt every time the
screen reads its settings. A rebuild that separates "declare the choices" from "read the
current value" must make the first idempotent.

## `OnBtnUpClick` / `OnBtnDownClick` / `CanPressUp` / `CanPressDown`

**Contract** — the cursor walks one entry at a time and stops at both ends; the predicates are
simply whether there is a next or previous index, which is what greys the arrows out at the
list's ends. The two click handlers move the cursor, repaint, and delegate to the base so the
owner learns of the click.

**Notes** — the base's two mutators, the ones the auto-repeat calls, are deliberately **empty**
here. A held arrow on a text spin box therefore does nothing at all: only discrete clicks move
it. For a list of half a dozen named choices that is the right call, and it means the
accelerating repeat never applies to this subtype.

## `SetItem` / `GetTokenText`

**Contract** — paint an entry's translated label; read the current entry's *original* name.
The asymmetry is the point: the world outside this widget only ever sees the untranslated
spelling.

## `SaveOptValue` / `SaveBackUpOptValue` / `UndoOptValue` / `IsChangedOptValue`

**Contract** — the rest of the options protocol, over the cursor index: save writes the current
entry's original name into the setting as text; back up and undo copy the index; the change
test compares indices.

**Notes** — the token's numeric identifier is carried through every entry and never consulted:
this widget saves by *name*, not by identifier. A rebuild may drop the field, but only after
confirming that no consumer of the setting expects the numeric form — the token mechanism
supports both, and other options items use the number.
