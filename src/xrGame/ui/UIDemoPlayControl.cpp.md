# src/xrGame/ui/UIDemoPlayControl.cpp

> The transport bar for recorded multiplayer demos: play/pause, speed, restart, a progress read-out, and a two-level "rewind until *this* happens to *that* player" menu.

**Needs** — [`UIDemoPlayControl.h`](UIDemoPlayControl.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [`xrUICore/ListBox/UIListBoxItem.h`](../../xrUICore/ListBox/UIListBoxItem.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`Level.h`](../Level.h.md) · [`DemoInfo.h`](../DemoInfo.h.md) · [`DemoPlay_Control.h`](../DemoPlay_Control.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md)
**Used by** — [`UIDemoPlayControl.h`](UIDemoPlayControl.h.md)
**Tier floor** — T3: menus, a clock read-out and command strings.

## Purpose

A recorded demo is a network capture replayed through the client. This screen is its
transport. Two things about it are worth a rebuilder's attention and the rest is layout:
**every transport action is issued as a console command rather than as a call into the
playback engine**, and **the rewind menu is a menu with a submenu whose selection tags carry
an off-by-one encoding**.

## State

```text
RECORD DemoTransport EXTENDS DialogScreen
  playback         : DemoPlayback          # resolved from the level once, at init
  rewind_type      : RewindKind            # last chosen predicate
  rewind_target    : text                  # last chosen player name; empty means "any"
  players          : list<text>            # names, in demo order
  type_menu        : PopupMenu             # the predicate list
  player_menu      : PopupMenu             # the submenu of names
  popup_bounds     : rect                  # the region popups are allowed to open within
  type_menu_origin : point                 # where the predicate list opens
  last_cursor_pos  : point                 # restored when the screen is reopened
```

**Invariants**

- `work_in_pause` is true. The screen must keep responding precisely *because* the usual
  reason to open it is that playback is paused.
- The player menu is constructed as the type menu's submenu, so choosing a predicate can
  chain into choosing a player without the screen sequencing it.
- A rewind is in flight or it is not; the speed and repeat buttons are disabled for exactly
  the duration of one.

## Construction and binding

**Contract** — the screen's children are created in code, then configured from a shipped
layout document by name, then each interactive child is bound to a handler. The layout
supplies geometry, textures and text for every element; the code supplies only the element
names, which are therefore **frozen** against the shipped XML. The playback object is
resolved from the running level and its absence is fatal — the screen cannot be opened
outside a demo.

Two positions are computed rather than authored, because they depend on measured sizes:

```text
type_menu_origin.x := background.left + background.width - type_menu.width - 14
type_menu_origin.y := background.top  - type_menu.height
```

That is: the predicate list is right-aligned inside the transport bar with a 14-unit inset,
and hangs *above* it. Hanging above is not cosmetic — the transport bar sits at the bottom of
the screen and a downward-opening menu would fall off the canvas.

**Notes** — an ordinary window is constructed, initialized from the layout, read for its
rectangle, and then discarded. Its only product is `popup_bounds`, the region popups may open
inside. The layout gets to author that region without it belonging to any live widget. A
rebuild reads the rectangle straight from the document instead.

## The rewind menu

**Contract** — the predicate list offers six entries: round start, a kill, a death, an
artefact taken, dropped, or delivered. Choosing *round start* rewinds immediately, because it
takes no subject. Choosing any other predicate opens the player submenu. Choosing a player
starts the rewind. Both lists are localized at build time and sized to their content.

The two lists encode their selection in the row tag, and the encodings differ:

```text
# predicate list: tag IS the predicate
# player list:    tag 0  == "any player"
#                 tag n  == players[n - 1]
```

**Notes** — the off-by-one exists so that zero can mean *any*. It is the kind of encoding a
rebuild silently gets wrong and then reads one player off in every rewind; it is called out
in the source with a shouted comment, which is the strongest evidence available that it has
bitten someone. A rebuild with an optional-typed tag should use one and delete the shift.

The player list is built from the demo's own header, which records who was in the match; it
is not the live player list, and a player who left mid-demo still appears.

## Starting a rewind

**Contract** — issue the chosen predicate and target to the playback engine with a
completion callback. A rewind that the engine refuses to start (no such event ahead in the
capture) changes nothing. A rewind that starts disables the two speed buttons and the repeat
button until it completes; the completion callback re-enables them. Any other transport
action cancels an in-flight rewind first.

```text
FUNCTION begin_rewind()
  cancel_any_rewind()
  started := playback.rewind_until(predicate_for(rewind_type), rewind_target, on_rewind_done)
  IF started THEN disable(increase_speed, decrease_speed, repeat_rewind)
```

**Notes** — "cancel first" is applied to *every* handler including play/pause and restart,
not just to a new rewind. Rewinding is a seek that runs across many frames; leaving one
running while the player also changes speed would race the seek against the transport.

`repeat_rewind` re-runs the last predicate and target without reopening the menus, which is
the whole reason the last choice is remembered as state rather than read from the menu.

## Transport actions

**Contract** — play/pause toggles the engine's own pause, giving a reason string that
appears in the log. Restart, speed-up and speed-down are issued as **console commands** by
name.

**Notes** — this is the chapter's rule that a UI action becomes a game action through an
event rather than a direct call, in its bluntest form: the screen names a command and the
console dispatches it. The benefit is that the same three actions are reachable from a key
binding, from a script and from the console with one implementation, and the screen holds no
knowledge of what "multiply speed" means. The cost is that the command names are frozen
strings shared with the shipped key-binding configuration. A rebuild keeps the indirection —
a named command registry — and is free to drop the textual console.

Pause is toggled directly rather than by command because the screen must also know the
current pause state to render its status line.

## Per-frame status

**Contract** — each update composes one localized status line — paused or playing, position
as a whole percentage, and speed as a one-decimal multiplier — and drives the progress bar
from the same position. Position is a fraction of the capture's length.

## Input

**Contract** — releasing the *crouch* binding hides the predicate menu, remembers the cursor
position, and closes the screen. Everything else falls through to the ordinary screen
dispatch.

**Notes** — crouch is a strange key to close a transport bar with, and there is no recoverable
reason for it beyond its being free during demo playback, where the player's avatar does not
exist. Remembering the cursor position and restoring it on the next open is the reason the
screen is closed-and-reopened rather than hidden: the transport bar is a cursor-grabbing
screen over a running replay, so the player toggles it constantly and expects the pointer to
be where they left it.
