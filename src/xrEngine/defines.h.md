# src/xrEngine/defines.h

> The global display mode, the global feature flag set, and the three game-identity switches.

**Needs** — [`defines.cpp`](defines.cpp.md)
**Used by** — [`Device_mode.cpp`](Device_mode.cpp.md) · [`defines.cpp`](defines.cpp.md) · [`device.cpp`](device.cpp.md) · [`device.h`](device.h.md) · [`profiler.h`](profiler.h.md) · [`stdafx.h`](stdafx.h.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) · [`particle_actions_collection_io.cpp`](../xrParticles/particle_actions_collection_io.cpp.md) · [`stdafx.h`](../xrParticles/stdafx.h.md)
**Tier floor** — T1: a bit set whose numbering is written into the user's settings file, and a mode record passed to the windowing layer

## Purpose

Three groups of process-wide state that nearly every file in the engine reads, gathered
here because putting them anywhere else would drag that file's dependencies along with them.

**Frozen by name, not by value.** The flags are set by name through console variables and
written back to the user's settings file, so a rebuild may renumber the bits but must keep
the names and the meanings.

## State

**Which game is running.** Three mutually exclusive switches, decided at startup by
inspecting the installed data, and read wherever behaviour differs between the three
commercial titles the engine supports.

```text
call_of_pripyat_mode   : bool
clear_sky_mode         : bool
shadow_of_chernobyl_mode : bool
```

**The display mode.** Handed to the windowing layer at creation and at every reset.

```text
RECORD DisplayMode
  monitor       : int     # index into the enumerated displays
  window_style  : WindowStyle
  width, height : int     # 0 means "ask the platform for the desktop's"
  refresh_rate  : int     # 0 means "the platform's choice"
  bits_per_pixel: int     # 32

ENUM WindowStyle
  WINDOWED               # an ordinary resizable window
  WINDOWED_BORDERLESS    # no border, still a window            <- the default
  FULLSCREEN_BORDERLESS  # borderless, topmost, scaled to the desktop: looks fullscreen
  FULLSCREEN             # exclusive fullscreen
```

**Notes** — the two "fullscreen" styles are genuinely different: the borderless one keeps
the desktop compositor and switches instantly, the exclusive one takes the display and can
change its mode. Both ship because each is wrong for somebody — exclusive fullscreen is
faster and breaks alt-tab, borderless is friendlier and adds a frame of latency.

**The feature flags.** One bit set, read everywhere.

```text
rsAlwaysActive          # keep running at full rate when the window loses focus
rsClearBB               # clear the back buffer each frame even when fully overdrawn
rsVSync
rsWireframe
rsConstantFPS           # feed the simulation a fixed timestep regardless of real time
rsDisableObjectsAsCrows # never demote a distant object to the cheap update path
rsStatistic             # the statistics overlay
rsCameraPos             # the camera-position readout
rsShowFPS
rsShowFPSGraph
rsOcclusionDraw         # visualize the occlusion culler
rsDrawStatic            # draw level geometry
rsDrawDynamic           # draw objects
rsDrawDetails           # draw the grass-and-debris layer
rsDrawParticles
mtSound                 # run the sound update on a worker
mtPhysics               # run physics on a worker
mtNetwork               # run the network update on a worker
mtParticles             # run particle simulation on a worker
# bits 20 and above are reserved for the editor application
```

**Invariants** — the four `mt` flags are *not* debug switches: they select which subsystems
are advanced off the main thread, and turning one off changes the order updates observe each
other in. A rebuild with a different threading model still needs the switches, because they
are the fallback when a subsystem's worker path misbehaves.

The reservation of the top twelve bits is a real constraint: the editor application sets
its own flags in the same word, so an engine that claims bit 20 breaks it.
