# src/xrGame/ui/UIDialogWnd.cpp

> The base every full-screen game screen derives from: it knows the holder that is showing it, decides whether input reaches it while the world is paused, and offers itself to the holder's open/close protocol.

**Needs** — [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`UIDialogHolder.h`](../UIDialogHolder.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIDialogWnd.h`](UIDialogWnd.h.md)
**Tier floor** — T3: a flag, a back-link and three branches.

## Purpose

Chapter 15 gives a widget tree; it does not give the notion of *a screen that is currently
up*. That notion — one of a stack of screens owned by a holder, which grabs the cursor,
suspends the world, and consumes input that would otherwise reach the player's hands — lives
here, and it is the single class every screen in this chapter descends from.

The file is tiny because the holder does the work. What this class contributes is the
**input-admission rule**, which is the one decision in it that a rebuild must copy exactly.

## State

```text
RECORD DialogScreen EXTENDS Window
  holder       : optional<DialogHolder>   # set by the holder on open, cleared on close
  work_in_pause: bool                     # default false
```

**Invariants**

- `holder` is non-empty exactly while the screen is on a holder's stack. Every close path
  goes through the holder, so a screen never clears its own link.
- The screen is created hidden. Being shown and being on the stack are separate facts; the
  holder sets both, in that order.

## Input admission

**Contract** — before a keystroke, a text character or a controller axis is offered to the
widget tree, the screen answers one question: *should this screen be receiving input at all
right now?* Three conditions, in order, and the order matters:

```text
FUNCTION admits_input() -> bool
  IF NOT enabled            THEN RETURN false   # a disabled screen is inert even while shown
  IF holder.ignores_pause   THEN RETURN true    # the holder overrides everything below
  IF world_is_paused AND NOT work_in_pause THEN RETURN false
  RETURN true
```

**Notes** — the holder's override comes *before* the pause test, not after, and that
ordering is the whole point. The main menu pauses the world and must still take input; it
gets that by being hosted in a holder that ignores pause, not by every menu screen setting
its own flag. A screen that wants input during a pause under an ordinary holder sets
`work_in_pause` instead. Two mechanisms for what looks like one condition, because they have
different scopes: one is a property of the host, one of the screen.

Keyboard and controller admission run the same test and then delegate to the ordinary widget
dispatch. The two channels are separate all the way down — see chapter 15's note that
scancodes and composed characters are distinct channels.

## `Show`

**Contract** — showing the screen also resets the whole subtree beneath it. A screen is
therefore reopened in its authored state rather than in whatever state the player left it,
which is why list scroll positions and highlighted rows do not survive a close/open cycle.
Hiding does not reset.

## `ShowDialog` / `HideDialog` / `ShowOrHideDialog`

**Contract** — the three ways a screen is toggled from outside. Opening asks the *currently
active* holder to start this screen, optionally suppressing the heads-up indicators while it
is up; closing asks *this screen's own* holder to stop it. The toggle form composes the two.
Opening an already-open screen and closing a closed one are both no-ops.

**Notes** — the asymmetry is deliberate and easy to get wrong in a rebuild. **Open goes to
whoever is active now; close goes to whoever opened it.** A screen may be started by the
game's holder and later find the menu's holder active; closing it must still unwind the stack
it is actually on.

The "hide indicators" flag is passed through to the holder, which is what dims the ammo
counter and the crosshair behind the inventory but not behind a quick message box.

## Screen-level defaults

**Contract** — four questions a holder asks a screen it is about to show, each with a
default a screen overrides only when it differs: does the player stop moving while this is up
(yes), does the pointer need to be visible (yes), should the pointer be re-centred on open
(yes), and does this screen keep running while the world is paused (no). A fifth,
`Dispatch`, lets a screen answer an engine command by number; the default accepts every
command and does nothing.

**Notes** — "stop any move" is not a UI property; it is the reason opening the inventory
does not leave the player walking. A rebuild that routes movement input through the same
consumption chain as UI input gets this for free and should still keep the query, because the
map screen answers it differently.

## Debug surface

**Contract** — the screen contributes its holder's identity and its pause flag to the
development inspector described at
[Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui). Excluded from
release builds.
