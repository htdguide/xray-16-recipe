# src/xrEngine/Engine.cpp

> Wires the whole engine together at startup: picks a renderer, brings up the game module, starts the scheduler, and puts the audio update and the event pump into the frame sequence at the right priorities.

**Needs** — [`Engine.h`](Engine.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`xrSheduler.h`](xrSheduler.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`xr_input.h`](xr_input.h.md) · [`device.h`](device.h.md) · [`xrSound`](../xrSound/README.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`Engine.h`](Engine.h.md); callers name that, not this file.
**Tier floor** — T2: composition and ordering. Nothing here touches a device directly.

## Purpose

One object owns the four services that must exist for the engine to run — the module loader, the event queue, the update scheduler and the audio manager — and this file is where they are started in the one order that works. It is also where the renderer is chosen, which is the single most consequential startup decision: it determines which shaders load, which level detail exists, and whether the game can run at all on this machine.

## State

```text
RECORD Engine
  external  : ModuleRegistry      # renderer and game module loading; see EngineAPI.cpp
  events    : EventQueue          # named events with deferred delivery; see EventAPI.cpp
  scheduler : UpdateScheduler     # the time-budgeted per-frame update; see xrSheduler.cpp
  sound     : AudioManager
  quit_event : event handle
```

A single process-wide instance. The four services are held by value rather than created on demand because their lifetimes are the process's and because circular construction between them would otherwise have to be broken by hand.

## `initialize`

**Contract** — Given the game module and the (at most two) renderer modules the executable was built or linked with, brings the engine to the state where a frame may run. Registers three frame handlers at deliberately chosen priorities, builds the renderer list, selects a renderer, initializes the game module and starts the scheduler. Fails hard if no renderer can be selected — see [`EngineAPI.cpp`](EngineAPI.cpp.md).

```text
FUNCTION initialize(game_module, renderer_modules)
  quit_event = events.attach("KERNEL:quit", self)

  frame_sequence.add(self,            priority = HIGH + 1000)   # pumps deferred events first
  frame_sequence.add(audio_update,    priority = NORMAL - 1000) # after the level's update
  frame_parallel_sequence.add(audio_render)                     # off the main thread

  external.build_renderer_list(renderer_modules)
  select_renderer()
  external.initialize(game_module)
  scheduler.initialize()
```

**Notes** — The three priorities are the load-bearing part. The engine's own handler runs *first* of everything, so deferred events raised last frame are delivered before any subsystem updates and every subsystem sees a consistent event set. The audio *update* — which moves the listener to the camera — runs at a priority below normal, explicitly "after the level update", because the camera does not reach its final position until the level has updated the actor and the camera manager has committed. Listening from last frame's camera position is audible. The audio *render* — refilling streaming buffers — runs on the parallel frame sequence because it is I/O-bound decode work with no dependency on the frame's state.

**Notes** — The audio update reads the committed camera basis as four vectors, position/direction/up/right, rather than a matrix. The audio seam wants a listener orientation and the engine has already orthonormalized the basis, so passing it whole avoids re-deriving it.

## `select_renderer`

**Contract** — Decides which renderer generation runs. A command-line switch wins outright; otherwise the choice is re-read from the user's settings file, and doing so marks the renderer as overridable so a later automatic downgrade is allowed. A dedicated server is forced to the oldest generation unconditionally.

```text
FUNCTION select_renderer()
  IF dedicated_server THEN
    console.run("renderer renderer_r1")     # no scene is drawn; use the cheapest to create
    RETURN
  FOR EACH switch IN [rgl, r4, r3, r2.5, r2a, r2, r1]   # order matters: see Notes
    IF command line contains "-<switch>" THEN
      console.run("renderer renderer_<switch>") ; RETURN
  re-execute only the "renderer" line of the user's settings file
  allow_renderer_override = true
```

**Notes** — The switch scan order is not alphabetical and not arbitrary: it is longest-prefix-first among names that share a prefix. `-r2.5` and `-r2a` must both be tested before `-r2`, or a substring match would claim them. This is a consequence of matching command-line switches as substrings rather than parsing them; a rebuild that tokenizes the command line properly can test in any order.

**Notes** — The seven names are frozen. They are what shipped configuration files and every modding guide say, and the settings file stores the choice by name. `r1` is the original forward renderer, `r2` through `r4` are successive deferred generations, `r2.5` and `r2a` are intermediate variants, and `rgl` is the OpenGL backend, which is a *renderer name* rather than an API selector because from the console's point of view a backend and a generation are the same kind of choice.

**Notes** — Re-executing one line of the settings file, rather than reading the console variable that the settings file already set, exists because the renderer must be chosen before most of the console exists. The settings file is parsed twice: once early for this one line, once fully later.

**Notes** — Taking the value from the settings file sets an override flag; taking it from the command line does not. The flag permits the renderer-selection step to silently fall back to a different generation when the requested one fails its hardware or data requirements. A player who named a renderer on the command line gets a hard failure instead, because they asked for something specific.

## `destroy`

**Contract** — Stops the scheduler, finalizes the game module and renderer, tears down the event queue, and removes the three frame handlers. Mirror of initialize, in reverse.

## `on_event`

**Contract** — Handles the one event the engine subscribes to itself: quit. Releases input capture and pushes a quit request onto the host's event queue rather than terminating. Going through the host queue is what makes quit uniform — a window close, a console command and a script call all end at the same place, and the loop exits from one condition.

## `on_frame`

**Contract** — The engine's own frame handler: delivers everything deferred onto the event queue since last frame. It runs before every other handler, so a subsystem raising an event during its own update is delivering it to next frame's start rather than mid-frame.
