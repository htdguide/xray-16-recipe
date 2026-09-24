# src/xrEngine/AccessibilityShortcuts.hpp

> Suppresses the desktop's accessibility interrupts and screen saver for the lifetime of a fullscreen session, and restores exactly what it found when the session ends.

**Needs** — [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`x_ray.cpp`](x_ray.cpp.md)
**Tier floor** — T1: it reads and writes desktop-environment settings through the host OS's own configuration interface, which is a foreign boundary with no portable equivalent.

## Purpose

A fullscreen game holds the keyboard for long, repetitive presses — the exact gesture the desktop's accessibility features interpret as a request to turn themselves on. Holding shift for eight seconds raises a sticky-keys prompt; repeated keys raise a filter-keys prompt; both steal focus and, in an exclusive fullscreen mode, can lose the graphics device. The screen saver is the same problem in slow motion: a player using only mouse and gamepad looks idle to the desktop.

This is a scoped guard: one object, created when the session begins and destroyed when it ends, that turns those features off and puts them back. It exists as a separate unit only because it is desktop-specific and everything else in the device layer is not.

## State

```text
RECORD AccessibilityGuard
  screensaver_was_enabled : bool
  sticky_saved            : settings_blob    # the desktop's own record, kept verbatim
  filter_saved            : settings_blob
  toggle_saved            : settings_blob
  sticky_saved_flags      : int              # invariant: 0 means "we did not touch it"
  filter_saved_flags      : int
  toggle_saved_flags      : int
```

The invariant that matters: a zero saved-flags field means the feature was never disabled by us and must not be re-enabled on the way out. Restoring a feature the user had switched off would be a worse bug than leaving it off.

## `disable`

**Contract** — Reads the desktop's current screen-saver and accessibility state, records it, then clears the enable bit on each feature that the desktop reports as *available*. Features the desktop does not offer are skipped entirely, so nothing is recorded for them and nothing is restored later. No failure path: if the desktop refuses, the game runs with the prompts enabled, which is a nuisance and not an error.

```text
FUNCTION disable()
  screensaver_was_enabled = query_screensaver_active()
  IF screensaver_was_enabled THEN set_screensaver_active(false)

  FOR EACH feature IN [sticky, filter, toggle]
    saved[feature] = query(feature)
    IF saved[feature] is marked available THEN
      saved_flags[feature] = saved[feature].flags   # non-zero: we own the restore
      write(feature, flags = 0)
```

## `release`

**Contract** — The mirror image, run unconditionally at the end of the session including on an unclean shutdown path that still unwinds scopes. Restores only what `disable` recorded as ours.

**Notes** — In the original this is a destructor, which is the whole reason the type exists rather than a pair of free calls: it guarantees the restore runs when the enclosing scope leaves by any route. A rebuild needs the same guarantee — a scope guard, a defer, a `finally` — and must additionally accept that a hard crash skips it, leaving the desktop with accessibility off until the user fixes it. That is the original's behaviour too.
