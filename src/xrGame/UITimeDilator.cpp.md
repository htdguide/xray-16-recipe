# src/xrGame/UITimeDilator.cpp

> Slows the simulation clock while certain menus are open, if the player asked for it.

**Needs** — [`UITimeDilator.h`](UITimeDilator.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two predicates over a flag set and one write to the clock's rate.

## Purpose

The inventory and the PDA can be opened mid-firefight. This decides what the world does
while they are: run at full speed, stop, or anything between — one factor applied to the
simulation's rate, with a per-menu opt-in so a player can slow time for the inventory but
not the PDA.

The object is a lazily created process-wide singleton reached through a named accessor.
A rebuild should hand it to the single-player game UI explicitly instead; nothing else
touches it.

## State

```text
RECORD TimeDilator
  factor        : real = 1.0      # the rate to run at while dilating
  enabled_modes : set<Mode>       # which menus the player opted in for
  current_mode  : Mode            # None, Inventory, or Pda

ENUM Mode: None, Inventory, Pda   # bit values, so a set is one small flag word
```

**Invariant** — the clock's rate is 1.0 whenever `current_mode` is `None` or is not in
`enabled_modes`. Every entry point re-establishes this rather than tracking whether it
has already been established.

## `SetCurrentMode`

**Contract** — called on every dialog open and close. Records which menu is now on top
and either starts or stops dilation.

```text
FUNCTION set_current_mode(d, mode)
  d.current_mode = mode
  IF mode <> None THEN start(d) ELSE stop(d)

FUNCTION start(d)
  IF d.current_mode <> None AND d.current_mode IN d.enabled_modes THEN
    simulation clock rate = d.factor

FUNCTION stop(d)
  simulation clock rate = 1.0
```

The asymmetry is deliberate: starting is conditional on the opt-in, stopping is
unconditional, so a mode turned off while its menu is open restores normal speed at once.

## `SetUiTimeFactor` · `GetUiTimeFactor`

**Contract** — sets the factor and immediately re-applies it, so a settings slider
changed with the menu open takes effect while the player watches.

## `SetModeEnability` · `GetModeEnability`

**Contract** — opts one menu in or out. Turning a mode on re-applies dilation; turning
*the currently open* mode off stops it. Turning a non-current mode off changes nothing
visible.

## `TimeDilator` · `CloseTimeDilator`

**Contract** — the accessor creates the object on first use and returns the same one
thereafter; the closer destroys it. The single-player game UI's teardown is what calls
the closer, which means the dilator's lifetime is really the level's.

**Notes** — nothing resets the clock rate when the dilator is destroyed. If the game UI
goes away while dilating, the rate stays where it was. Any rebuild should restore 1.0 on
teardown.
