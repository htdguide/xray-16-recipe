# src/xrGame/fire_disp_controller.cpp

> Smooths the crosshair's spread: when the weapon's dispersion changes, the displayed value travels to the new one over time instead of jumping.

**Needs** — [`fire_disp_controller.h`](fire_disp_controller.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`Level.h`](Level.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one linear interpolation against the wall clock

## Purpose

Weapon dispersion changes in steps — the player stops moving, crouches, aims, fires — and a
crosshair that snapped to each new value would flicker. This tracks a current displayed
dispersion that eases toward the target, at a rate the active weapon chooses.

It is a separate object rather than a field on the crosshair because the value must persist
across the many places that set dispersion, and because the easing duration depends on the
weapon rather than on the crosshair.

## State

```text
RECORD FireDispertionController
  start_disp   : real      # where the current transition began
  end_disp     : real      # where it is going
  start_time   : real (s)  # global clock at the start of the transition
  current_disp : real      # what the crosshair draws
```

Invariant: `current_disp` lies between `start_disp` and `end_disp` while a transition runs,
and equals `end_disp` once it has finished. There is no "finished" flag — the time
comparison is the state.

## `SetDispertion`

**Contract** — sets a new target dispersion. If the target is effectively unchanged, only
refreshes the current value; otherwise begins a new transition from wherever the display
currently is. Always ends by recomputing the current value, so the caller can read it
immediately.

```text
FUNCTION set_dispertion(new_disp)
  IF new_disp is not ~= end_disp
    start_disp = (start_disp is ~= 0) ? new_disp : current_disp     # see Notes
    end_disp   = new_disp
    start_time = global_clock
  update()
```

**Notes** — the choice of the new starting point is the one subtle line. Normally a new
transition starts from the value on screen right now, so an interrupted ease continues
smoothly from where it got to. But when `start_disp` is still zero — the very first call,
before anything has been displayed — it starts from the *target*, which makes the first
transition instantaneous. That is right: the crosshair should appear at its correct spread
when the weapon is drawn, not sweep in from zero.

The guard is on `start_disp` rather than on a dedicated first-call flag, which means a
weapon whose dispersion legitimately eases to exactly zero will make its *next* change
instantaneous too. Harmless in practice and a wart a rebuild should not copy: use an
explicit "not yet initialised" state.

## `Update`

**Contract** — recomputes the displayed dispersion from the wall clock. Reads the player's
active weapon each call to get the easing rate, falling back to a default when there is no
player or no weapon. Clamps to the target once the transition's duration has elapsed. Pure
apart from writing the current value; no allocation.

```text
FUNCTION update()
  inertion = default_inertion
  IF the controlled entity is the player AND it has an active weapon
    inertion = weapon.crosshair_inertion

  duration = inertion * abs(end_disp - start_disp)     # see Notes
  end_time = start_time + duration
  IF duration is zero OR global_clock > end_time
    current_disp = end_disp
    RETURN
  current_disp = start_disp
               + (end_disp - start_disp) * (global_clock - start_time) / duration
```

**Notes** — the duration is proportional to the *size* of the change, which makes the easing
a constant rate rather than a constant time: the crosshair always opens and closes at the
same speed on screen, and a big change simply takes longer. The weapon's inertion value is
therefore "seconds per unit of dispersion", and the default of 5.91 is documented in the
source as the time to traverse one full unit. That number has no derivation beyond feel; it
is only a fallback, since every weapon supplies its own.

Reaching through the level to the current entity, to its inventory, to its active weapon, on
every update, is incidental plumbing — the controller has no back-reference to its owner. A
rebuild should hand it the rate, not make it go looking.

The controller is driven by the player's weapon specifically. It is a HUD device: there is
one crosshair and it belongs to whoever the camera is attached to.
