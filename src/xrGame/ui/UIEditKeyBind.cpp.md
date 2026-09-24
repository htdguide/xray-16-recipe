# src/xrGame/ui/UIEditKeyBind.cpp

> One editable key-binding cell: it captures the next key pressed, announces the capture to every other cell on the page so duplicates can stand down, and commits as a console binding command.

**Needs** — [`UIEditKeyBind.h`](UIEditKeyBind.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Options/UIOptionsItem.h`](../../xrUICore/Options/UIOptionsItem.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UIEditKeyBind.h`](UIEditKeyBind.h.md) · [`UIListItemServer.cpp`](UIListItemServer.cpp.md)
**Tier floor** — T3.

## Purpose

The controls page is a table of (action, primary key, secondary key) — or (action, gamepad
button). This is one cell of it, and it is both a widget and a settings control implementing
chapter 15's four-step protocol (read, back up, commit, undo) against a binding rather than
a console variable.

Three decisions live here. **Capture**: a cell in edit mode swallows the next input event
and treats it as the value. **Conflict**: a cell that takes a key announces it and every
other cell holding that key decides for itself whether to give it up. **Commit**: the
binding is applied by issuing a console command, so the same path serves the screen, the
console and a configuration file.

## State

```text
RECORD KeyBindCell EXTENDS Label, SettingsControl
  slot        : { primary_keyboard, secondary_keyboard, gamepad }
  action      : GameAction          # resolved from the settings entry name
  key         : optional<Key>       # the current value
  backup_key  : optional<Key>       # the value at page open
  editing     : bool
```

**Invariants**

- `slot` is fixed at construction and never changes; it selects which of the action's three
  binding slots this cell reads and writes.
- **A cell edits its own input kind only.** A keyboard cell refuses gamepad buttons and a
  gamepad cell refuses keys, silently, by consuming the event and doing nothing. Without
  that, a stray controller nudge while a keyboard cell is armed would bind it.
- An empty binding displays as three dashes, never as blank — a blank cell is
  indistinguishable from a missing row.

## Edit mode

**Contract** — clicking a cell with the primary button arms it. While armed it takes
**keyboard capture from its parent** so the whole page's input funnels to it, turns its
background texture on, and runs a cyclic alpha animation on its text colour so the player can
see which cell is waiting. Losing focus disarms it, restores full text alpha and releases
capture.

**Notes** — the blinking is a named colour animation from the game's own animation library,
applied to the text colour with only the alpha channel taken and looped. Reusing the world's
light-animation curves for a UI blink is the original's shortcut; what matters to a rebuild is
that the armed state is *visible and continuous*, not a static highlight.

## Taking a key

**Contract** — while armed, the next accepted input becomes the value:

```text
FUNCTION on_input(code) -> consumed
  IF NOT editing THEN RETURN not consumed
  IF kind_of(code) does not match this cell's kind THEN RETURN consumed   # refuse quietly
  key := lookup_key(code)
  IF key is unknown THEN RETURN consumed                                  # unnamed key: ignore
  display key's localized name
  disarm
  broadcast "<action name>=<key name>" to the settings group this cell belongs to
  RETURN consumed
```

**Invariants** — mouse buttons arrive on the pointer channel, not the key channel, so the
cell takes them in its pointer handler and **explicitly refuses mouse codes on the key
handler** to keep one press from being bound twice. Chapter 15's separation of channels is
what makes this necessary and what makes it sufficient.

Gamepad axes arrive on a third channel and are taken the same way, so an axis can be bound
like a button.

**Notes** — the value is stored as the *key*, and it is displayed under its **localized**
name but broadcast and committed under its **canonical** name. Two names for one key: one
for the player, one for the configuration file. A rebuild that keeps only the localized name
writes a configuration file that will not load in another language.

## Conflict resolution by broadcast

**Contract** — when a cell takes a key it sends `"<action>=<key>"` to every other settings
control in its group. Each recipient decides alone:

```text
FUNCTION on_peer_bound(message)
  parse message into (other_action, key_name)
  IF this cell has no key                        THEN RETURN
  IF key_name is not this cell's key             THEN RETURN
  IF other_action is this cell's own action      THEN RETURN   # our own echo
  IF the two actions' key groups do not conflict
     AND their key contexts do not conflict      THEN RETURN   # both may hold the key
  clear this cell's key and show three dashes
```

**Notes** — this is the whole conflict model and it is **peer-to-peer, not central**. No
registry of who holds what; the page is the registry, and each cell answers for itself.
The consequence a rebuild must preserve: conflicts are only resolved among cells *currently
on the page*, so a binding held by an action the page does not show is never disturbed.

The two conflict predicates are why a key can legitimately be bound twice. Actions belong to
a **key group** (which input mode they belong to — on foot, in a vehicle, in a menu) and a
**key context** (which screen is up). Two actions that can never be live at the same moment
may share a key, and the shipped configuration relies on that heavily. The predicates
themselves live with the input layer; see
[`xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md).

A cell also receives its own broadcast and must recognise it — the action-name test is the
guard, and it has to come *after* the key test, otherwise an unrelated cell holding the same
key would be compared against the wrong action.

## The four-step settings protocol

**Contract** —

- *read* — take the current key from the action's binding slot chosen by this cell's slot,
  and display it;
- *back up* — remember the key as it was when the page opened;
- *commit* — issue the binding as a console command: one of three command names by slot,
  with the action's canonical name and the key's canonical name;
- *undo* — restore the remembered key;
- *changed?* — the current key differs from the backed-up one.

**Notes** — commit goes through the console rather than writing the binding table directly.
That is the same indirection the demo transport uses and for the same reason: one
implementation of "bind this action to this key", reachable from the screen, from a
configuration file and from the console, with the screen holding no knowledge of the binding
table's shape. Three command names rather than one command with a slot argument is an
artefact of the shipped configuration files, which contain those names verbatim.

An undone cell is *not* re-committed — undo restores the displayed value, and the binding
table was never touched, because commit is the only writer.

## Text fitting

**Contract** — a key name too wide for the cell is shortened until it fits, and the cell
never wraps or scrolls.

```text
FUNCTION fit(text, width) -> text
  IF the font is multi-byte
    THEN RETURN text cut at the font's own reported character position for that width
    ELSE drop trailing characters one at a time until the measured width fits
```

**Notes** — the two branches exist because the shipped fonts come in two kinds and only one
of them can be indexed by character. Chapter 15 states the same split for wrapping; this is
the same constraint reaching one more place. Dropping characters one at a time is a linear
re-measure, acceptable because it runs once per value change on a short string.

## Appearance

**Contract** — the cell's background is chosen by probing the texture registry for three
names in order, taking the first that exists.

**Notes** — the three names belong to the three shipped games, whose UI data ships different
row textures under different names. Probing rather than configuring is how one executable
dresses itself for whichever game's data is mounted. The order matters: the third name exists
in two of the games, so it must be tried last. This is a recurring pattern in this chapter and
a rebuild should keep it as an explicit *first-of* lookup rather than a per-game branch.
