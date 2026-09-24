# src/xrGame/DBG_Car.cpp

> The vehicle's developer overlay — a live readout of the drivetrain and three
> plots of the engine's power and torque curves against revolutions.

**Needs** — [`Car.h`](Car.h.md) · [`Level.h`](Level.h.md) · [`PHDebug.h`](PHDebug.h.md) · [`Hit.h`](Hit.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrEngine/StatGraph.h`](../xrEngine/StatGraph.h.md) · [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md) · [`xrUICore/ui_base.h`](../xrUICore/ui_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure diagnostics drawing text and line plots.

## Purpose

Tuning a vehicle means answering "where on the torque curve is the engine right now, and did
the transmission shift where I wanted it to". This file draws that answer. It exists as a
separate file, and compiles to nothing outside a debug build, purely so the vehicle's real
logic is not interleaved with instrumentation.

A rebuild may omit this file entirely. It is documented because the *quantities* it chooses
to display are a useful statement of what the drivetrain model considers to be its own state.

## State

```text
RECORD CarDebug                       # lives inside the car, debug builds only
  power_rpm    : function plot        # engine power against revolutions, static curve
  torque_rpm   : function plot        # engine torque against revolutions, static curve
  dynamic_plot : rolling plot         # three live traces scrolling in time
  plots_built  : bool
```

**Invariants** — the overlay is only built for the car the player is currently *looking
through*. A car being driven by someone else, or an unoccupied car, draws nothing, and
losing the view entity tears the plots down. Without that gate the overlay would be drawn
once per vehicle in the level.

## the scheduled check

**Contract** — run from the car's scheduled update: if the plot debug flag is set and this
car is both physically alive and the current view entity, build the plots; otherwise destroy
them. Building and destroying are idempotent, so this can run every tick.

## building the plots

**Contract** — sample the car's own power and torque functions across its declared
revolution range into two static curves, mark them up, and allocate one rolling plot for the
live traces.

```text
FUNCTION build_plots()
  IF plots_built THEN RETURN
  force the drive state to "driving" for the duration          # see Notes
  power_rpm  = sample(car.power_curve,  min_rpm .. max_rpm)
  torque_rpm = sample(car.torque_curve, min_rpm .. max_rpm)    # power / rpm
  give each five markers: current rpm, current value, rpm implied by the wheels,
                          and the current gear's two shift thresholds
  IF the transmission shifts automatically AND the all-gears flag is set
      FOR EACH gear beyond the first
          mark its up-shift and down-shift revolutions on both curves
  dynamic_plot = three scrolling traces: power, torque, revolutions
  restore the drive state
  plots_built = true
```

**Notes** — the drive state is forced to "driving" while sampling because the car's power
function reports reduced output in neutral. Sampling it in whatever state the car happens to
be in would plot a different curve depending on whether the player's foot was down.

The two live traces that are not power are each divided by a precomputed ratio so all three
fit one vertical axis. The ratios are captured once at build time from the curve's own
maximum, which is why they are stored rather than recomputed: the axis must not rescale while
the traces scroll, or the shape of the trace would be unreadable.

Gear shift points are drawn on both curves in a matched colour pair so that the up-shift and
down-shift revolutions can be compared against the torque peak at a glance. That comparison
is the single most common thing a vehicle tuner needs to see.

## the live readout

**Contract** — drawn each frame for the viewed car. Speed in kilometres per hour, the
current gear, its ratio, engine power, revolutions per minute, torque at the reference wheel,
torque at the engine, remaining fuel, and a coloured flag line for each of: clutch engaged,
engine running, stalling, starter turning, brakes applied.

**Notes** — the displayed figures are converted out of the engine's internal units at the
display site: speed from units-per-second to kilometres per hour, revolutions from radians
per second to revolutions per minute, power divided by a scale factor. That the conversions
live here and not in the model is deliberate — internally the drivetrain works in one
coherent set of units and only the human-readable layer knows about the others.

The gear number is drawn in red while a shift is in progress, which is the cheapest possible
way to see that the transmission spends a visible amount of time between gears.
