# src/xrGame/script_particles_script.cpp

> Exports the standalone particle effect to the script layer as `particles_object`.

**Needs** — [`script_particles.h`](script_particles.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of a script-owned particle effect.

## `script_register`

**Contract** — registers a class named `particles_object`, constructed from an effect name,
with:

```text
play()                       # start where it currently is
play_at_pos(position)        # place it, then start
stop()                       # cut emission and kill live particles
stop_deffered()              # stop emitting; let live particles finish
stop_deferred()              # the same function, spelled correctly

playing()  -> bool
looped()   -> bool

move_to(position, velocity)
set_direction(vector)
set_orientation(yaw, pitch, roll)
last_position() -> vector

load_path(name)
start_path(looped)
stop_path()
pause_path(paused)
```

**Notes**

`stop_deffered` and `stop_deferred` are the same function under two names. The misspelling
shipped first and scripts use it; the correct spelling was added beside it rather than
replacing it. A rebuild must export both.

The handle is script-owned: the effect dies when the script drops its last reference, with
no engine-side registry to look it up in. That is what makes this type usable for a
one-off effect in a cutscene and unusable for anything that must survive a save.
