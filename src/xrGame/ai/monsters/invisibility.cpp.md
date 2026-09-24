# src/xrGame/ai/monsters/invisibility.cpp

> An energy budget that drains while a creature is hidden and refills while it is shown, with a burst of flicker covering each transition so the switch is seen rather than instantaneous.

**Needs** — [`invisibility.h`](invisibility.h.md) · [Seam: Windowing and input](../../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — reached through its declarations in [`invisibility.h`](invisibility.h.md); callers name that, not this file.
**Tier floor** — T3: one scalar integrated against frame time and two timestamps compared against a global clock

## Purpose

Two behaviours that look like one. The **budget** decides how long a creature may stay
invisible: it falls at a tuned rate while hidden and rises at the same rate while visible, so
"how long can it hide" and "how long before it can hide again" are one number. The creature's
brain reads the budget to decide whether hiding is affordable.

The **flicker** is what the player sees. Rather than swapping visibility in one frame, each
transition opens a window during which the rendered visibility toggles back and forth at a
tuned interval, and only when the window closes does the visibility settle at whatever the new
state demands. The flicker is driven by two timestamps against the global clock, not by a
countdown, so it is unaffected by how often the frame update actually runs.

Whether the creature is *meant* to be hidden and whether it is *currently rendered* are
separate facts. The first is the state; the second is the flicker's output. Only the second
reaches the creature, through a callback fired on every toggle.

## State

```text
RECORD Invisibility
  active            : bool      # intent: the creature means to be hidden
  energy            : real      # invariant: within [0, 1]
  manual            : bool      # script has taken the budget away from the simulation

  blinking          : bool      # a transition window is open
  blink_started_at  : int       # global clock when the window opened
  last_toggle_at    : int       # global clock at the last visibility flip; 0 means none yet
  rendered_visible  : bool      # the flicker's current output

  # authored in the creature's configuration section
  blink_duration    : int       # "Invisibility_BlinkTime", milliseconds the window stays open
  blink_interval    : int       # "Invisibility_BlinkMicroInterval", milliseconds between flips
  energy_rate       : real      # "Invisibility_EnergySpeed", budget units per second
```

Invariants: `energy` never leaves the unit interval — every write clamps. While `manual` is
set, nothing changes `energy` at all, so a scripted hide is unbounded in time. `blinking`
false implies `rendered_visible` already agrees with `active`.

## `reload`

**Contract** — reads the three tuned numbers from the named configuration section. All three
are required; there are no defaults, so a creature that inherits this mixin and omits them
fails to load. Called once when the creature loads its section.

## `reinit`

**Contract** — resets to visible, empty budget, no flicker, no manual control. Called on spawn
and whenever the creature is re-created from its record. Note that the budget starts *empty*,
not full: a creature cannot vanish the instant it appears, it must first charge.

## `activate` / `deactivate`

**Contract** — set or clear the intent to be hidden, and open a flicker window. Both are
idempotent: calling `activate` while already active does nothing at all, not even a flicker.
Each notifies the creature through its `on_activate` / `on_deactivate` hook *after* the state
has changed, so the hook may read the new state.

```text
FUNCTION activate()
  IF active THEN RETURN          # idempotent: no second flicker
  start_blink()
  active = true
  creature.on_activate()
```

**Notes** — the flicker window is opened *before* the state flips, which matters only in that
the first toggle after the flip is computed against the new intent.

## `frame_update`

**Contract** — called once per rendered frame. Advances the flicker window, then integrates
the budget against frame time unless script has taken manual control. Allocates nothing,
blocks never.

```text
FUNCTION frame_update()
  update_blink()

  IF manual THEN RETURN          # script owns the budget; it neither drains nor charges
  IF active
    energy = energy - energy_rate * frame_seconds
  ELSE
    energy = energy + energy_rate * frame_seconds
  clamp energy into [0, 1]
```

**Notes** — drain and charge share one rate, so the duty cycle a creature can sustain is
exactly half. A rebuild that wants asymmetric hide and recharge times needs a second number
the original does not have.

## `update_blink` — the transition window

**Contract** — a private step of `frame_update`. While a window is open, flips the rendered
visibility every `blink_interval` milliseconds of global clock and notifies the creature on
each flip. When `blink_duration` has elapsed since the window opened, closes it and settles
the rendered visibility at the negation of the intent — hidden means not rendered.

```text
FUNCTION update_blink()
  IF NOT blinking THEN RETURN
  now = global_clock

  IF blink_started_at + blink_duration < now
    blinking = false
    rendered_visible = NOT active           # settle: intent decides the resting value
    creature.on_change_visibility(rendered_visible)
    RETURN

  IF last_toggle_at + blink_interval < now
    last_toggle_at = now
    rendered_visible = NOT rendered_visible
    creature.on_change_visibility(rendered_visible)
```

**Notes** — `last_toggle_at` is reset to zero, not to the current clock, when a window opens.
Against a global clock that is far past zero this guarantees the first toggle fires on the very
next update, so a window always begins with a flip rather than with an interval of stillness.
That is the reason for the zero and it is worth keeping.

The window closes on elapsed time, not on a toggle count, so the number of flips a transition
shows is `blink_duration / blink_interval` rounded by whatever the update rate happens to be —
it is not a fixed count and the two numbers are authored independently.

## Manual control

**Contract** — `set_manual_control` hands the budget to script. While manual: `frame_update`
leaves the budget alone entirely, and `manual_activate` / `manual_deactivate` are the only
calls that take effect — each is a no-op unless manual is set. `activate` and `deactivate`
themselves remain callable and still work, so manual control gates *the manual entry points*,
not the underlying transition.

**Notes** — the asymmetry is real: taking manual control freezes the budget but does not lock
out the creature's own brain from hiding. A rebuild that reads this as a lock will diverge.
