# src/xrEngine

This is the spine. Every other chapter is either something this one calls, something it
drives, or a service it reaches for. It owns the window and the graphics device, the clocks,
the frame loop, the input layer, the console, the registry of everything alive on a level,
the scheduler that advances those things within a time budget, the level's own lifecycle
and loading screen, the weather and time-of-day system, the camera, the event queue and the
profiler. It does **not** own the renderer, the physics, the interface toolkit, the AI or
the game: it declares the interfaces those live behind and calls them.

Read this chapter as the answer to two questions: *what happens between the process
starting and the first image reaching the screen*, and *what one iteration of the loop
does*. Everything here is in service of one of those two.

## Where it sits

Chapter 13 of the build order. It rests on the platform layer
([`src/Common`](../Common/README.md)), the containers and math
([`src/xrCommon`](../xrCommon/README.md),
[`src/utils/xrMiscMath`](../utils/xrMiscMath/README.md)), the interfaces
([`src/Include`](../Include/README.md)), the global environment struct
([`src/Layers/xrAPI`](../Layers/xrAPI/README.md)), the virtual filesystem and configuration
parser ([`src/xrCore`](../xrCore/README.md)), the collision database
([`src/xrCDB`](../xrCDB/README.md)), the material table
([`src/xrMaterialSystem`](../xrMaterialSystem/README.md)), the network transport
([`src/xrNetServer`](../xrNetServer/README.md)), the script virtual machine
([`src/xrScriptEngine`](../xrScriptEngine/README.md)), the audio system
([`src/xrSound`](../xrSound/README.md)) and the particle simulation
([`src/xrParticles`](../xrParticles/README.md)).

Three chapters link *back* against it and are driven by it —
[`xrPhysics`](../xrPhysics/README.md), [`xrUICore`](../xrUICore/README.md) and
[`xrAICore`](../xrAICore/README.md). That is the first of the three cycles the system
requirements name, and it is broken at the interface: this chapter declares abstract ports
(`IPhysicsShell`, `ILoadingScreen`, `IGameFont`, `IPHdebug`) and those chapters supply the
adapters. A rebuild should make the direction explicit and keep the ports here.

The renderer and the game module are loaded as **modules**, selected by name at startup.
Neither is linked: the engine knows only the interfaces in
[`Render.h`](Render.h.md) and [`EngineAPI.h`](EngineAPI.h.md).

---

## Part 1 — From process start to the first frame

The order below is a dependency graph, not a narration. Each step names what makes it
impossible to move earlier or later. The code is in [`x_ray.cpp`](x_ray.cpp.md) and
[`Engine.cpp`](Engine.cpp.md).

1. **Read the dedicated-server switch, install the crash handler, start the windowing
   library.** Nothing above this line may read a game file, because there is no filesystem
   yet. The splash image is therefore compiled into the executable, not loaded
   ([`embedded_resources_management.h`](embedded_resources_management.h.md)).
2. **Raise the splash**, on its own thread, and suppress the desktop's accessibility
   interrupts ([`AccessibilityShortcuts.hpp`](AccessibilityShortcuts.hpp.md)) — both because
   bring-up takes seconds with nothing on screen.
3. **Start two background tasks**: create the input layer, and enumerate the audio output
   devices. Both are slow, both are needed later, and both are independent of everything
   between here and where they are awaited.
4. **Mount the virtual filesystem.** The hinge of the whole sequence. Everything below may
   read game data.
5. **Parse the configuration set** and decide which of the three shipped games' conventions
   are in force ([`defines.h`](defines.h.md)). One of three process-wide flags is set, and
   the engine branches on it at every behavioural divergence between the games.
6. **Bring up the debug overlay toolkit and enumerate the display modes**
   ([`Device_imgui.cpp`](Device_imgui.cpp.md), [`Device_mode.cpp`](Device_mode.cpp.md)).
7. **Await the input layer, then create the console** and register every command
   ([`XR_IOConsole.cpp`](XR_IOConsole.cpp.md), [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md),
   [`xr_level_controller.cpp`](xr_level_controller.cpp.md)). The console needs input,
   because it registers key bindings.
8. **Select a renderer and load its module** ([`EngineAPI.cpp`](EngineAPI.cpp.md)). A
   command-line switch wins outright; otherwise one line of the settings file is re-read for
   it, which also permits an automatic fallback if the named renderer cannot run here.
9. **Initialize the device**: create the window, choose a display mode, start the clocks
   ([`Device_Initialize.cpp`](Device_Initialize.cpp.md)).
10. **Execute the default key bindings, then the user's settings file.** In that order —
    defaults first, so an incomplete settings file still leaves a playable key map.
11. **Await the audio device list and open the audio device** the settings named.
12. **Execute any `start` or `load` on the command line** as a console line. The console is
    the engine's only command surface; the launcher reaches it the same way a script does.
13. **Create the graphics device**: swap chain, render targets, the material library
    ([`Device_create.cpp`](Device_create.cpp.md)).
14. **Ask the game module for its persistent layer** and tell it the application started
    ([`IGame_Persistent.cpp`](IGame_Persistent.cpp.md)). That call loads the weather set and
    scans the installed levels. With no game module, the console is shown instead and the
    engine is a shell.
15. **Hide the splash, calibrate the real clock against the platform's tick counter, show
    the window, and enter the loop.**

A level does not exist yet. The first frames the loop runs are console or main-menu frames.
A level arrives only when a `start` command is executed — see Part 3.

---

## Part 2 — One iteration of the loop

The loop is in [`x_ray.cpp`](x_ray.cpp.md); the frame is in [`device.cpp`](device.cpp.md).

```text
drain up to 32 window events; collapse the main window's focus transitions to one
process one frame
service the presence client
```

The frame itself:

```text
gate            is there a device? is a level still loading? (if so, run one load step
                and draw the loading screen instead of a frame)
update phase    advance the clocks, then broadcast the per-frame signal in priority order
camera          rebuild the view, projection and derived matrices from the camera
                the update phase just moved
render phase    concurrently with: the deferred one-shot closures and the worker-side
                update sequence
                ask the backend to begin; broadcast the render signal; draw the statistics
                overlay and the debug overlay; end, which presents
limiter         sleep out the remainder of the frame's budget
```

Four things about this shape are decisions a rebuild must make too.

**The engine is not pipelined.** Simulation finishes before drawing starts. What runs
*beside* drawing is only the leftovers explicitly marked safe to: the deferred closures and
the worker-side half of the update sequence. The camera matrices are captured into a
*saved* set at the end of the render phase precisely because those workers outlive the
frame they belong to.

**The frame delta is smoothed and clamped, not fixed.** Ninety percent of the new
measurement plus ten percent of the old, clamped to a tenth of a second. The clamp is the
important half: a frame that really took longer — a streaming hitch, a breakpoint, a laptop
waking — is reported as a tenth of a second, so the world runs in slow motion rather than
teleporting. **The fixed timestep the system requirements describe is not here**; it is in
the simulation layer, which consumes this delta and accumulates its own fixed increments
against it. The engine's *pacing* is variable; the world's *advance* is not.

**There are three clocks and the distinction is load-bearing.** Two pause with the game and
scale with the time factor — one drives the frame delta, one is the scriptable global time.
The third never pauses and never scales: it is what the network layer, the streaming audio
and the text editor's key repeat are timed against, because a menu or an alt-tab must not
look to them like time standing still. A fourth derived value, *continual* time, is real
time minus every millisecond the window spent unfocused.

**The limiter is two caps.** In game, effectively uncapped; in a menu or paused, hard at 60,
because a menu renders in microseconds and would otherwise spin a core.

---

## Part 3 — Bringing a level up and taking it down

Starting, loading, disconnecting and playing back a recorded match are **four deferred
events**, never direct calls, because every one of them destroys or creates the level from
underneath whatever asked ([`EventAPI.cpp`](EventAPI.cpp.md),
[`IGame_Persistent.cpp`](IGame_Persistent.cpp.md)).

A start resolves the level's folder, re-points the `$level$` logical filesystem root at it,
mounts that level's archive if it is not already mounted, and then runs the load
([`IGame_Level.cpp`](IGame_Level.cpp.md)): the chunked level file, the render geometry, the
collision database, the visibility topology, the navigation mesh, the spawn records, and
finally the script layer's own load.

**Loading is a queue of resumable steps, not a blocking call.** One step runs per iteration
of the main loop, each reporting whether it has finished, so the window keeps pumping events
and the loading screen keeps drawing. Work that genuinely cannot be split calls the
*other* path — a single frame drawn from inside the middle of the load, by opening a render
bracket by hand. A rebuild with coroutines has a better tool for both; the requirement is
only that the platform's event queue keeps being served.

**After the load comes the precache**, measured in *frames* rather than seconds. The camera
is spun through a full circle with the renderer in deferred-load mode and the sound muted,
so that every shader, texture and mesh the level can show is requested and uploaded before
the player is given control; everything not touched during the spin is then dropped, because
the set reached by a full spin is the level's real working set. This is why the first ten
seconds of a level do not stutter.

Tearing down is the reverse and is checked: no entity may be registered twice, and a
destroyed entity must be unreferenced by the scheduler, the render graph and the physics
world before its memory is released. That is what
[`pure_relcase.cpp`](pure_relcase.cpp.md) exists for.

---

## Ideas to hold before reading the twins

**Signals, not calls.** Nine named per-frame and per-lifecycle broadcasts, each with a
priority-ordered subscriber list ([`pure.h`](pure.h.md)). A subsystem joins one to be driven.
The signal set is *closed*: a module cannot invent a new one, so the frame loop is readable
as a fixed sequence, and the only question a reader ever has is "at what priority". The
three that matter most: the update phase, the render phase, and the device-reset signal that
tells every holder of a device resource to rebuild it. One signal is *exclusive* — only its
highest-priority subscriber runs — which is how a console that has the keyboard, or a video
that owns the frame, takes it away from everything else.

**Events, not signals, when the recipient must not run now.** A named event queue with
reference-counted identity and deferred delivery, drained once at the top of every frame
([`EventAPI.cpp`](EventAPI.cpp.md)). Anything that resets the device, unloads the level or
quits goes through here, because none of those may run from inside the code that asked.

**Everything configurable is a console command, and the names are frozen.** The console is
not a debug prompt — it is the engine's configuration surface, its persistence layer and
its command API all at once. Shipped configuration files, the user's settings, the launcher
and every modification's scripts address these names, so a rebuild may reimplement a command
and may not rename one ([`XR_IOConsole.cpp`](XR_IOConsole.cpp.md),
[`xr_ioc_cmd.h`](xr_ioc_cmd.h.md)). A variable binds directly to the storage the rest of the
engine reads, with no indirection at the read site and no change notification.

**Input is bound by scancode and dispatched by action.** Nothing in the engine reads a key;
everything reads an *action* — `jump`, `use`, `ui_accept` — and the binding layer is the only
place that knows which physical key produces which
([`xr_level_controller.cpp`](xr_level_controller.cpp.md)). Bindings are stored by scancode so
they survive a keyboard-layout change, and each key carries three names: the scancode that
is compared, the internal name that appears in settings files, and the display name the
platform supplies for the player. Two orthogonal filters — a *group* (single-player,
multiplayer, both) and a *context* (none, interface, map, conversation) — let one key mean
different things in different situations without any reader knowing.

**Only the top of the input stack hears anything.** Receivers are a stack; pushing one
delivers a synthetic release burst to the one below so it does not think a key is still held
([`IInputReceiver.cpp`](IInputReceiver.cpp.md)).

**Two update paths, and choosing between them is a real decision.** Objects that need
unconditional per-frame work are updated by the object registry
([`xr_object_list.cpp`](xr_object_list.cpp.md)), parent before child. Everything else is
*scheduled* ([`xrSheduler.cpp`](xrSheduler.cpp.md)): a priority queue keyed by when each
object is next due, drained under a **time budget** rather than to completion, with each
object choosing its own rate by reporting how much it currently matters. The budget itself
is a control loop that grows when work is missed and shrinks when it is not, settling where
it overruns about one frame in four. A level has thousands of things wanting periodic work
and can afford none of them every frame; this is how both facts are true at once.

**Single player is a networked session against a local server.** There is exactly one
session path: a single-player game is a server and a client in one process. Every level,
every entity and every save is shaped by that.

**Time of day is an interpolation between two authored frames.** The weather system holds a
sequence of keyed frames and continuously blends the two the clock currently falls between,
producing the sky, fog, sun direction, ambient colour and wind the renderer and the audio
system read ([`Environment.cpp`](Environment.cpp.md)). A *weather effect* — a storm — is
spliced into that sequence rather than replacing it. Everything visually atmospheric in the
game is downstream of this one blend.

**The global environment struct is the engine's service locator.** Modules reach their peers
through one mutable global filled at startup. The recipe notes at each use site what is
actually being reached for; a rebuild should make it explicit injection.

---

## Twins

Grouped by what they do, not by filename. Every file in the chapter that has a twin is
listed. The chapter also contains build files and four plain-text design notes
(`ClientServer.txt`, `Effects description.txt`, `features.txt`, `TODO.txt`) which are not
part of the program and have no twins.

### The device and the frame

| File | Role |
|---|---|
| [`device.cpp`](device.cpp.md) | The frame loop: what happens between one presented image and the next, and the clocks everything else reads. |
| [`device.h`](device.h.md) | Declares the device: the window, the clocks, the camera, the frame loop and every per-frame callback list. |
| [`Device_Initialize.cpp`](Device_Initialize.cpp.md) | Brings the window into existence before anything else, and starts the two clocks the whole engine reads. |
| [`Device_create.cpp`](Device_create.cpp.md) | Creates the graphics device behind the existing window, loads the material library, declares the device ready. |
| [`Device_destroy.cpp`](Device_destroy.cpp.md) | Tears the graphics device down, and rebuilds it in place when the mode changes or the device is lost. |
| [`Device_mode.cpp`](Device_mode.cpp.md) | Display-mode enumeration and every window geometry decision: monitor, resolution, borders, fullscreen. |
| [`Device_imgui.cpp`](Device_imgui.cpp.md) | Stands the debug overlay toolkit up against the engine's allocator, window and clipboard. |
| [`Device_overdraw.cpp`](Device_overdraw.cpp.md) | A dead hook for the overdraw visualization mode. |
| [`defines.cpp`](defines.cpp.md) | The startup values of the global display mode and feature flags. |
| [`defines.h`](defines.h.md) | The global display mode, the feature flag set, and the three game-identity switches. |

### Process bring-up and module wiring

| File | Role |
|---|---|
| [`x_ray.cpp`](x_ray.cpp.md) | Process bring-up in order, the platform event pump that feeds the frame loop, and the shutdown that reverses it. |
| [`x_ray.h`](x_ray.h.md) | Declares the application object: construct it, call it once, destroy it. |
| [`Engine.cpp`](Engine.cpp.md) | Wires the engine together: picks a renderer, brings up the game module, starts the scheduler, places the audio update in the frame sequence. |
| [`Engine.h`](Engine.h.md) | Declares the composition root and the render-context count. |
| [`EngineAPI.cpp`](EngineAPI.cpp.md) | Enumerates the renderers the build offers, picks one this machine can run, hands the game module its object factory. |
| [`EngineAPI.h`](EngineAPI.h.md) | Declares the module boundaries: the object factory, the game module, the renderer module. |
| [`embedded_resources_management.h`](embedded_resources_management.h.md) | Gets the splash image and window icon into the process without a filesystem. |
| [`AccessibilityShortcuts.hpp`](AccessibilityShortcuts.hpp.md) | Suppresses the desktop's accessibility interrupts and screen saver for a fullscreen session, and restores what it found. |
| [`mailSlot.cpp`](mailSlot.cpp.md) | A debug-only one-way text channel by which an external tool feeds console commands into a running engine. |
| [`mp_logging.h`](mp_logging.h.md) | The build switch for multiplayer trace logging, plus one magic number used as a debug token. |
| [`stdafx.h`](stdafx.h.md) | The prelude every engine translation unit opens with, and the one place fixing the order of the layers. |
| [`stdafx.cpp`](stdafx.cpp.md) | The compilation anchor for that prelude. |

### Signals and events

| File | Role |
|---|---|
| [`pure.h`](pure.h.md) | The broadcast mechanism — nine named signals and the priority-ordered subscriber list they are delivered through. |
| [`pure.cpp`](pure.cpp.md) | The compilation anchor for the signal registry. |
| [`pure_relcase.cpp`](pure_relcase.cpp.md) | The base that makes any non-object holder of object references get told before those references go stale. |
| [`pure_relcase.h`](pure_relcase.h.md) | Declares the relcase subscriber base. |
| [`EventAPI.cpp`](EventAPI.cpp.md) | A registry of named events with reference-counted identity, immediate or deferred delivery, drained at the top of every frame. |
| [`EventAPI.h`](EventAPI.h.md) | Declares the named-event queue and what a receiver must implement. |

### Input

| File | Role |
|---|---|
| [`xr_input.cpp`](xr_input.cpp.md) | The input layer — one flat key space over keyboard, mouse and gamepad, a receiver stack, and the per-frame translation into press/hold/release. |
| [`xr_input.h`](xr_input.h.md) | Declares the unified key space, the controller state record and the input layer. |
| [`IInputReceiver.cpp`](IInputReceiver.cpp.md) | Stack push and pop for an input receiver, and the synthetic release burst when one loses the input. |
| [`IInputReceiver.h`](IInputReceiver.h.md) | The interface anything wanting input implements, plus every input tuning value the player can set. |
| [`xr_level_controller.cpp`](xr_level_controller.cpp.md) | The binding layer: the frozen action table, the frozen key table, the three-slot map between them, and the commands that edit it. |
| [`xr_level_controller.h`](xr_level_controller.h.md) | Declares the action vocabulary every input consumer speaks, and the shape of a binding. |
| [`key_binding_registrator_script.cpp`](key_binding_registrator_script.cpp.md) | Exports the action vocabulary, the key contexts and every scancode to the script layer, by name. |

### Console and text entry

| File | Role |
|---|---|
| [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) | The console: the registry of every named variable and command, the line that executes them, and the suggestion list. |
| [`XR_IOConsole.h`](XR_IOConsole.h.md) | Declares the console: the typed variable registry, the command line, and the tip list. |
| [`XR_IOConsole_callback.cpp`](XR_IOConsole_callback.cpp.md) | What the arrow and tab keys mean inside the console's edit field, and how the buffer is rewritten under the cursor. |
| [`XR_IOConsole_control.cpp`](XR_IOConsole_control.cpp.md) | The two cursors — history and suggestion selection — and the clamping that keeps them inside their lists. |
| [`XR_IOConsole_get.cpp`](XR_IOConsole_get.cpp.md) | Reading a console variable back out with its type and bounds, for code that wants the setting rather than the command. |
| [`XR_IOConsole_script.cpp`](XR_IOConsole_script.cpp.md) | The console as scripts see it: run a line, read a setting, show or hide, defer execution to a safe moment. |
| [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) | The typed console-variable vocabulary: what a command is, and the seven kinds of bounded variable. |
| [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) | The engine's own command set, and the one function that registers every engine-level name. |
| [`line_edit_control.cpp`](line_edit_control.cpp.md) | A single-line text editor: selection, word motion, clipboard, one-step undo, and key auto-repeat driven by the frame clock. |
| [`line_edit_control.h`](line_edit_control.h.md) | Declares the text editor, its modifier vocabulary and its four modes. |
| [`edit_actions.cpp`](edit_actions.cpp.md) | The handler chain behind one key: try each link's modifier requirement, run the first that matches. |
| [`edit_actions.h`](edit_actions.h.md) | Declares the three kinds of link in that chain. |
| [`Text_Console.cpp`](Text_Console.cpp.md) | A console drawn with the operating system's own text drawing, because a dedicated server has no renderer. |
| [`Text_Console.h`](Text_Console.h.md) | Declares the dedicated server's console. |
| [`Text_Console_WndProc.cpp`](Text_Console_WndProc.cpp.md) | The two platform message handlers for the dedicated server's console windows. |
| [`StringTable/`](StringTable/README.md) | Every piece of player-visible text, keyed by identifier, in the configured language. |

### The debug overlay

| File | Role |
|---|---|
| [`editor_base.cpp`](editor_base.cpp.md) | The overlay's shell: a three-state visibility model, a self-registering tool list, and the menu bar that drives both. |
| [`editor_base.h`](editor_base.h.md) | Declares the shell and the interface a tool must satisfy to live in it. |
| [`editor_base_input.cpp`](editor_base_input.cpp.md) | The platform backend: event translation, the cursor, the text-input mode, and each tool's settings. |
| [`editor_helper.cpp`](editor_helper.cpp.md) | One widget behaviour the toolkit does not provide — a collapsing header whose state survives a changing parent. |
| [`editor_helper.h`](editor_helper.h.md) | The adapter between the engine's vocabulary and the toolkit — scoped identifiers, engine-typed widgets, scancode translation. |

### Objects, scheduling and the game module seam

| File | Role |
|---|---|
| [`xr_object.h`](xr_object.h.md) | The contract every client object satisfies — the widest interface in the engine, and the seam across which the engine drives the game. |
| [`xr_object_list.cpp`](xr_object_list.cpp.md) | The registry of every client object on the level — who exists, who is updated this frame, who is being destroyed, who answers to a network id. |
| [`xr_object_list.h`](xr_object_list.h.md) | Declares the per-level object registry. |
| [`xrSheduler.cpp`](xrSheduler.cpp.md) | The time-budgeted update scheduler — advances as many objects as it can afford, at rates each chooses, adapting its budget to the load. |
| [`xrSheduler.h`](xrSheduler.h.md) | Declares the scheduler. |
| [`ISheduled.cpp`](ISheduled.cpp.md) | Default bounds, registration, and the two guards that catch a scheduled object outliving or re-entering its own update. |
| [`ISheduled.h`](ISheduled.h.md) | What an object must answer to be advanced by the scheduler, and the bounds it declares. |
| [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) | Creates a client object from its configuration section, and warms the model and texture caches for a game type. |
| [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md) | Declares the client-object factory and its prefetch list. |
| [`IRenderable.cpp`](IRenderable.cpp.md) | Construction, destruction and lazy lighting-cache creation for anything drawable. |
| [`IRenderable.h`](IRenderable.h.md) | What an object must answer before the renderer will put it in a frame. |
| [`ICollidable.cpp`](ICollidable.cpp.md) | Registers a collidable entity into the spatial database, and owns its shape. |
| [`ICollidable.h`](ICollidable.h.md) | Declares "this entity has a collision shape", and the default filling of it. |
| [`ObjectDump.cpp`](ObjectDump.cpp.md) | Renders a game object as text, so a crash message can say what broke rather than where. |
| [`ObjectDump.h`](ObjectDump.h.md) | Declares the six object-to-text dumps. |
| [`vis_common.h`](vis_common.h.md) | The per-visual visibility record — the bounds a culler tests and the frame stamps that stop it being tested twice. |
| [`vis_object_data.h`](vis_object_data.h.md) | The per-visual parameters handed to material passes. |

### Level and session lifecycle

| File | Role |
|---|---|
| [`IGame_Level.cpp`](IGame_Level.cpp.md) | The level's lifecycle: load geometry, collision and objects in the one order that works; tear it down; distribute every emitted sound. |
| [`IGame_Level.h`](IGame_Level.h.md) | Declares the level: the global handle every subsystem reaches the world through, and the contract the game module fills. |
| [`IGame_Level_check_textures.cpp`](IGame_Level_check_textures.cpp.md) | Reports the loaded level's texture budget, and the limits it was authored against. |
| [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) | The layer that outlives any one level, and the driver of the start / load / disconnect lifecycle. |
| [`IGame_Persistent.h`](IGame_Persistent.h.md) | Declares that layer, the hooks the game module fills, and the main-menu interface. |
| [`ILoadingScreen.h`](ILoadingScreen.h.md) | The port through which the engine drives the loading screen it does not own. |
| [`CustomHUD.cpp`](CustomHUD.cpp.md) | Defines the heads-up display's default feature set. |
| [`CustomHUD.h`](CustomHUD.h.md) | The interface through which the frame loop drives the game's heads-up display, plus the HUD feature toggles. |

### The renderer boundary

| File | Role |
|---|---|
| [`Render.h`](Render.h.md) | The frame-graph boundary: everything the engine may ask of a graphics backend, and the three resource kinds it holds handles to. |
| [`Render.cpp`](Render.cpp.md) | The three pieces of that interface with behaviour: two self-releasing resource handles and a scoped context switch. |
| [`ShadersExternalData.h`](ShadersExternalData.h.md) | The handful of values the game pushes into shader constants without going through the material system. |
| [`EnnumerateVertices.h`](EnnumerateVertices.h.md) | A callback shape for walking a mesh's vertices without materializing them. |
| [`Properties.h`](Properties.h.md) | A self-describing stream of named, typed editor properties — the dialect the shader and effect tools speak. |

### Camera, effectors and demos

| File | Role |
|---|---|
| [`CameraManager.cpp`](CameraManager.cpp.md) | Runs the two effector stacks over the frame's camera description and writes the result into the device as view, projection and post-process parameters. |
| [`CameraManager.h`](CameraManager.h.md) | Declares the camera manager and the module-wide post-process reference values. |
| [`CameraBase.cpp`](CameraBase.cpp.md) | Loads a camera's authored rotation limits and reports how close an angle is to one. |
| [`CameraBase.h`](CameraBase.h.md) | The abstract camera: an owner, a yaw/pitch/roll orientation with optional limits, and a basis the game may set directly. |
| [`CameraDefs.h`](CameraDefs.h.md) | The camera's value types: the per-frame description effectors mutate, and the identity of the two effector stacks. |
| [`Effector.h`](Effector.h.md) | A camera effector: something with a lifetime that perturbs the frame's camera and then expires. |
| [`Effector.cpp`](Effector.cpp.md) | Empty. |
| [`EffectorPP.h`](EffectorPP.h.md) | A post-process effector: a timed contribution to colour grading, blur, grain and duality. |
| [`EffectorPP.cpp`](EffectorPP.cpp.md) | The default post-process effector: count down, contribute nothing. |
| [`FDemoPlay.cpp`](FDemoPlay.cpp.md) | Drives the camera along a recorded path and measures the frame rate while it does — the engine's benchmark. |
| [`FDemoPlay.h`](FDemoPlay.h.md) | Declares the demo-playback camera effector. |
| [`FDemoRecord.cpp`](FDemoRecord.cpp.md) | A free-flying camera that records its own keyframes, and the three screenshot modes built on it. |
| [`FDemoRecord.h`](FDemoRecord.h.md) | Declares the demo-recording camera. |

### Weather, sky and atmospheric effects

| File | Role |
|---|---|
| [`Environment.cpp`](Environment.cpp.md) | The weather clock: which two authored time-of-day frames the world is between, how a weather effect is spliced in, and the per-frame interpolation. |
| [`Environment.h`](Environment.h.md) | Declares the weather system: a frame, the blend of two, a local override volume, an ambient set, and the environment that owns them. |
| [`Environment_misc.cpp`](Environment_misc.cpp.md) | What a weather frame contains, how two blend, where the sun is, and how the set is read from and written back to configuration. |
| [`Environment_render.cpp`](Environment_render.cpp.md) | The four points in the frame at which the weather draws itself, and the rebuild after a device reset. |
| [`Environment_editor.cpp`](Environment_editor.cpp.md) | The in-game weather editor: live editing with immediate feedback and a save back to shipped-format configuration. |
| [`Rain.cpp`](Rain.cpp.md) | Rain as the player experiences it: drops born around the camera, each ray-traced once to find where it lands. |
| [`Rain.h`](Rain.h.md) | Declares the rain effect: a camera-following drop field, a splash pool and the ambient bed. |
| [`thunderbolt.cpp`](thunderbolt.cpp.md) | The lightning effect, and the push of its flash back into the sky, sun and fog colours. |
| [`thunderbolt.h`](thunderbolt.h.md) | Declares the lightning effect and the palettes weather frames select from. |
| [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md) | The sun's lens flare — five rays measure occlusion, the weather picks the flare, the result is one blend factor and a gradient. |
| [`xr_efflensflare.h`](xr_efflensflare.h.md) | Declares the lens-flare effect and the authored sun descriptions. |
| [`perlin.cpp`](perlin.cpp.md) | Classic gradient noise in one, two and three dimensions — the source of the wind gusts, flicker and drift. |
| [`perlin.h`](perlin.h.md) | Declares the coherent-noise generators. |
| [`xrHemisphere.cpp`](xrHemisphere.cpp.md) | Three baked, evenly-distributed hemisphere tessellations — the sky dome's mesh, and a fixed set of light directions. |
| [`xrHemisphere.h`](xrHemisphere.h.md) | Declares the baked hemisphere accessors. |
| [`LightAnimLibrary.cpp`](LightAnimLibrary.cpp.md) | The library of named colour animations — the answer to "make this light flicker like a campfire". |
| [`LightAnimLibrary.h`](LightAnimLibrary.h.md) | Declares the colour-animation library and one named animation. |
| [`WaveForm.h`](WaveForm.h.md) | A five-shape periodic function with amplitude, offset, phase and frequency — how authored data says "oscillate this". |

### Senses

| File | Role |
|---|---|
| [`Feel_Vision.cpp`](Feel_Vision.cpp.md) | Decides what an entity can see: a frustum query, a set difference against last frame, and one cached, transparency-aware ray per candidate. |
| [`Feel_Vision.h`](Feel_Vision.h.md) | Declares sight: a potentially-visible set, fuzzy per-target visibility that builds and decays, and the ray cache that makes it affordable. |
| [`Feel_Touch.cpp`](Feel_Touch.cpp.md) | Maintains "which entities are inside my radius right now", with enter and leave edges and a temporary exclusion list. |
| [`Feel_Touch.h`](Feel_Touch.h.md) | Declares the proximity-contact sense. |
| [`Feel_Sound.h`](Feel_Sound.h.md) | The interface by which an entity is told that a sound it could hear has been emitted. |

### Collision and the physics ports

| File | Role |
|---|---|
| [`xr_collide_form.cpp`](xr_collide_form.cpp.md) | The per-object collision proxies a ray can hit — per-bone primitives rebuilt each frame, an authored trigger volume, and a hand-built shape list. |
| [`xr_collide_form.h`](xr_collide_form.h.md) | Declares those proxies and the query vocabulary. |
| [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md) | Turns "the ray entered this bone's volume" into "the ray hit this bone's mesh, here". |
| [`cf_dynamic_mesh.h`](cf_dynamic_mesh.h.md) | Declares the collision form that refines a skeleton hit to the exact triangle. |
| [`IPhysicsShell.h`](IPhysicsShell.h.md) | The read-only view of a rigid-body assembly: its pose, its elements, their velocities and shapes. |
| [`IPhysicsGeometry.h`](IPhysicsGeometry.h.md) | One collision shape, asked only for its oriented bounding box and whether water touches it. |
| [`IObjectPhysicsCollision.h`](IObjectPhysicsCollision.h.md) | The read-only window through which a non-physics module asks an object for its physical body. |
| [`IPHdebug.h`](IPHdebug.h.md) | The port the physics module draws its debug triangles through. |
| [`phdebug.cpp`](phdebug.cpp.md) | Holds the process-wide handle to that debug renderer. |

### Particles and rigid animation

| File | Role |
|---|---|
| [`PS_instance.cpp`](PS_instance.cpp.md) | The life and death of one particle effect in the world, including why it cannot be deleted where it dies. |
| [`PS_instance.h`](PS_instance.h.md) | Declares the base every live particle effect derives from. |
| [`ObjectAnimator.cpp`](ObjectAnimator.cpp.md) | Plays an authored rigid-body motion for anything that moves without a skeleton. |
| [`ObjectAnimator.h`](ObjectAnimator.h.md) | Declares the rigid-object animator: a bank of named transform motions with one playing. |

### Text, images and video

| File | Role |
|---|---|
| [`GameFont.cpp`](GameFont.cpp.md) | Loads a bitmap font's character table from any of four authored layouts, measures text including expanded key bindings, and wraps it. |
| [`GameFont.h`](GameFont.h.md) | Declares the bitmap-atlas font. |
| [`IGameFont.hpp`](IGameFont.hpp.md) | The text-drawing interface every subsystem draws through, so debug overlays and the game's screens share one font. |
| [`xrImage_Resampler.cpp`](xrImage_Resampler.cpp.md) | Separable, filtered image rescaling — the engine's only general-purpose resampler. |
| [`xrImage_Resampler.h`](xrImage_Resampler.h.md) | Declares the filter menu and the rescale entry point. |
| [`xrTheora_Stream.cpp`](xrTheora_Stream.cpp.md) | One Theora video track inside an Ogg container — headers, a scanned frame index, decode-to-a-time with key-frame pre-roll. |
| [`xrTheora_Stream.h`](xrTheora_Stream.h.md) | Declares one Theora video track. |
| [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md) | A playable clip — one or two tracks, the playback clock, looping, and conversion into something the device can sample. |
| [`xrTheora_Surface.h`](xrTheora_Surface.h.md) | Declares the playable video clip. |
| [`tntQAVI.cpp`](tntQAVI.cpp.md) | Plays a legacy AVI clip as an animated texture, with a second clip supplying the alpha channel. |
| [`tntQAVI.h`](tntQAVI.h.md) | Declares the legacy AVI texture player. |

### Statistics and profiling

| File | Role |
|---|---|
| [`Stats.cpp`](Stats.cpp.md) | The statistics overlay: the fixed order in which every subsystem is asked to account for its frame. |
| [`Stats.h`](Stats.h.md) | Declares the overlay and the four spare timers every developer reaches for. |
| [`StatGraph.cpp`](StatGraph.cpp.md) | A scrolling graph of the last N values of something, to *see* a frame-rate problem rather than read about it. |
| [`StatGraph.h`](StatGraph.h.md) | Declares the scrolling graph and its shapes. |
| [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md) | Draws one performance complaint in big red text without disturbing the caller's text cursor. |
| [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) | Declares the one shipped filling of the alert port. |
| [`IPerformanceAlert.hpp`](IPerformanceAlert.hpp.md) | The port a statistics dump writes through when a number it printed is bad enough to shout about. |
| [`profiler.cpp`](profiler.cpp.md) | Turns a frame's raw timing samples into a sorted, indented tree of named rows on the debug overlay. |
| [`profiler.h`](profiler.h.md) | Declares the hierarchical CPU-timing profiler and the scoped sample it is fed by. |
| [`profiler_inline.h`](profiler_inline.h.md) | The scoped sample's two ends — take the clock on entry, submit the elapsed span on exit. |
