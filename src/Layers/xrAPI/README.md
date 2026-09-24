# src/Layers/xrAPI — the global service environment

Three files, one declaration, and the entire wiring diagram of the engine.

This chapter owns a single mutable record — call it the **environment** — that holds one
reference per major service: the renderer, the debug renderer, the draw-utility helpers,
the UI renderer, the render factory, the script engine, the AI space, the sound manager,
the UI core, and a flag saying whether this process renders at all. Every module in the
engine reads it; ten sites in five other modules write it. It is the engine's service
locator and, as `SYSTEM-REQUIREMENTS.md` §7 records, a deliberate cycle-breaker: the
renderer, the game, the AI layer, the sound layer and the UI layer all need each other,
and none of them may name another at link time.

## Where it sits

Chapter 5, and it rests on nothing. That is the point. The module that defines the
environment has one dependency — the project-wide compilation prelude, for the
export/import spelling — so every other module in the build can depend on *it* without
inheriting a dependency on any peer. Every module the environment names is compiled later
and depends on this one; none of them are named here in a way that requires their code,
only a promise that a type by that name exists.

The prelude that every translation unit in the project compiles against includes this
chapter's declaration. The consequence is important and easy to miss when reading any
later chapter: **a use of the environment is never announced by an import.** A file that
reaches across the whole engine looks exactly like a file that reaches nowhere.

## The ideas you need before the twins make sense

**Modules are separately linked libraries.** The engine can be built as one static image or
as a host plus loadable modules, and the loadable build is the one the design is shaped
for. The renderer is picked at startup from two candidates after a hardware probe; the
game layer is a module the executable hands to the engine. Neither can be linked into the
other, because the choice is made at run time. Yet the engine's frame loop must call the
renderer every frame, the renderer must ask the game for materials, the UI must ask the
renderer for quads, and the AI must call into Lua. The environment is how a call crosses
that boundary: the caller reads a reference out of a record it *can* see, and calls
through it.

**Nothing in the record is owned by the record.** It is ten borrowed references. Some point
at objects with static storage inside the filling module (the renderer's five slots), some
at heap objects the filler created and will destroy (script engine, UI core), one at an
object that registers *itself* from its own constructor (the AI space). The record never
allocates and never frees.

**There is no access control and no synchronization.** Reads are direct field reads from
any thread. One slot is filled on a worker thread and published only by the join with that
worker; a rebuild that keeps the shape must keep that join, because the record supplies no
ordering of its own.

**A too-early read is a crash, not a diagnostic.** The record starts zero-valued, so an
unfilled slot reads as "absent". Out of roughly 1,200 uses across the engine, fewer than
twenty check. Every other use dereferences blind. This is a decision, not an oversight: the
engine treats "the renderer is missing while a frame is being drawn" as a programming
error that should fail loudly and immediately, and it is right that a silent fallback would
be worse. It is also the reason the fill order below is load-bearing rather than
incidental — the order *is* the contract.

## The slots, their fillers, and the order they are filled in

| Slot | What it reaches | Filled by | Point in startup | Cleared |
|---|---|---|---|---|
| `isDedicatedServer` | whether this process renders and plays sound at all | the application, from the command line | step 1 — the very first decision, before the crash handler, the window system or the virtual filesystem exist | never |
| `Sound` | the 3D audio mixer and its device list | the sound manager, while enumerating output devices | step 2 — started as a background task before the filesystem is mounted, joined in step 5 | at shutdown, before the mixer is destroyed |
| `Render`, `RenderFactory`, `DU`, `UIRender`, `DRender` | the selected graphics backend and its four companion surfaces | the selected renderer module, in one step | step 3 — inside engine initialization, after the console exists (it holds the chosen renderer's name) and before the graphics device is created | never, in practice — see below |
| `UI` | widget-toolkit singletons: fonts, cursor, focus, the scissor stack | the game's persistent object, at application start | step 4 — after the graphics device is created and the frame loop is ready | at shutdown, third |
| `AISpace` | navigation graphs, path search, patrol paths | the AI space object itself, from its own base constructor | step 5 — lazily, the first time anything touches the AI space singleton | at shutdown, first (from the same object's destructor) |
| `ScriptEngine` | the Lua virtual machine and the whole binding surface | the AI space's initialization, immediately after registering itself | step 5, same moment; **skipped entirely on a dedicated server** | at shutdown, first, with the AI space |

Read as a sequence:

```text
FUNCTION startup(command_line)
  env.isDedicatedServer = command_line contains "-dedicated"   # step 1: everything downstream branches on this
  install crash handler                                        # already reads isDedicatedServer to pick silent mode
  SPAWN device_list_task: env.Sound = create_sound_manager()   # step 2: fills a slot from a worker
  mount virtual filesystem; read configuration
  create console                                               # the console owns the "renderer" setting
  select_renderer()                                            # step 3: fills five slots at once
  create graphics device
  AWAIT device_list_task                                       # the only thing that publishes env.Sound to readers
  open the audio device
  env.UI = create_ui_core()                                    # step 4
  # step 5 happens later, on first access to the AI space:
  #   AISpace registers itself from its constructor,
  #   then, unless dedicated, creates the script engine and boots the shipped scripts
```

**What is legal to read before each fill.** `isDedicatedServer` is legal from the first
instruction, because it is a flag with a meaningful zero (a normal client). Every other
slot is illegal to read before its filler runs and there is no way to ask. The engine's
own answer is a convention, not a check: code that runs during startup knows its own
position in the sequence, and code that runs during a frame knows every slot is filled
because a frame cannot begin before step 4. The handful of guarded sites mark the real
exceptions, and each one is a place where an object can be destroyed *after* the renderer
is gone or *before* it has arrived — a light or a glow releasing itself during teardown, a
collision-space object constructed in a tool build with no renderer, a level unloading
after the device has shut down. A rebuild that makes the slots optional and forces every
reader to handle absence will find those are the only places where absence is a real state
rather than a bug.

**The dedicated server fills the renderer slots anyway.** A rendering-free process still
selects a renderer module and still calls its setup, because the resource manager, the
material system and the visibility structures are all reached through the renderer
interface even when nothing is drawn; the `isDedicatedServer` flag is then consulted about
two dozen times *inside* the renderer to skip texture upload, shader compilation, state
changes and statistics. The flag is not "is there a renderer", it is "does anything reach
the graphics device". A rebuild that wants a genuinely headless server should split that
interface instead of threading one boolean through the whole backend.

**Renderer selection.** The console's stored `renderer` setting names a mode; if that mode
is absent or its module reports its requirements unmet (the commonest reason being that
the shipped shader set for that backend is not in the game data), the engine walks the
candidate list and takes the first that accepts, rewriting the console setting so the
choice sticks. Hardware probing happens once, when the candidate list is built, and is
described as slow enough to be worth doing exactly once. Only after a candidate is settled
does it fill the five slots — so a failed probe leaves the environment untouched rather
than half-filled.

## Lifetime at shutdown

Teardown is not the reverse of startup, and the difference is deliberate in two places and
accidental in a third.

```text
FUNCTION shutdown()
  game.on_app_end()
    destroy main menu and loading screen        # these read env.UI, so they go first
    destroy game globals
      destroy ai space                          # clears env.AISpace, then env.ScriptEngine
    destroy env.UI                              # slot cleared in the same step as the object
  destroy the persistent game object
  destroy input, settings, console
  destroy the sound manager                     # clears env.Sound FIRST, then destroys the mixer
  destroy the graphics device
  destroy the engine                            # drops its reference to the renderer module
                                                # -- the five renderer slots are never cleared
```

Three rules are worth carrying into a rebuild:

1. **A slot is cleared before the service behind it dies, not after.** The sound manager
   clears its slot as the first statement of its teardown, then stops and frees the mixer.
   Any other order gives every reader in the process a window in which the slot looks
   present and points at a corpse.
2. **Destroy-and-clear is one step where the environment owns the reference.** The script
   engine, the AI space and the UI core are destroyed through an operation that nulls the
   reference it was given, so there is no moment when the slot is stale.
3. **The renderer's five slots are never cleared.** The renderer module implements a
   clearing operation — it exists, it is correct, it checks that the slots still point at
   *this* module before nulling them — and nothing ever calls it. The engine drops its own
   reference to the module and stops. In practice nothing reads the renderer after the
   graphics device is destroyed, so the dangling references are never followed; but this is
   the one place in the chapter where the invariant is maintained by luck. A rebuild should
   call the clearing step, and should assert at process exit that every slot is empty.

There is no mechanism for replacing a filled slot at run time. The renderer cannot be
switched without restarting the process, and the script engine's "restart" rebuilds the
virtual machine's contents behind a reference that never moves — which is why script
restart broadcasts a pair of events (reset, started) so that every holder of a Lua handle
can drop and re-acquire it. The reference is stable; what it points at is not.

## Why this exists, and what a rebuild should do instead

It exists because a call must cross a boundary that the build system refuses to let a
dependency cross. Remove the constraint — a renderer chosen at run time, a game layer
loaded as a module, a UI that draws through whichever backend won — and the record's whole
justification goes with it.

A rebuild has two honest replacements.

**Explicit dependency injection.** Each module declares the ports it needs as construction
parameters; the frame loop is handed a renderer, the UI is handed a UI-renderer, the AI is
handed a script runtime. The compiler then enforces the startup order that is presently a
convention, and a too-early read becomes a program that does not build rather than a crash
in a shipped binary. The cost is real and should not be understated: roughly 1,200 use
sites stop being a bare field read and start being a member the enclosing object must be
given, which means constructors and call chains grow parameters all the way down — the
sound manager is reached from deep inside entity code, the renderer from deep inside the
UI. Over half of those sites reach the script engine alone and most of the rest reach the
renderer or the UI renderer, so the three ports that carry this cost are known in advance
and should be resolved once per object rather than looked up once per call.

**A composition root.** One place builds every service in dependency order and hands each
its collaborators; nothing else constructs anything. This keeps the single wiring diagram
the environment gives you today — genuinely useful, because *one file answers the question
"what does this engine consist of"* — while deleting the ambient global. It costs the
lazy initialization the current design leans on: the AI space and the script engine are
created on first touch, part-way through a level load, and a composition root must either
create them eagerly at startup (paying their cost in a process that may never load a
level, which matters for the dedicated server, which skips the script engine entirely) or
hold them behind explicit lazy cells, which is the current behaviour with its ordering made
visible.

Either way, keep one property of the original: the **fill order** documented above is the
engine's real startup contract, and it should survive as an ordered construction sequence
that a reader can see in one place. What should not survive is that the contract is
enforced by everyone remembering it.

## Files

| Twin | Role |
|---|---|
| [`xrAPI.cpp`](xrAPI.cpp.md) | Defines the one process-wide instance of the environment record |
| [`stdafx.h`](stdafx.h.md) | Compilation prelude; notable only because the project-wide prelude it pulls in makes the environment visible everywhere |
| [`stdafx.cpp`](stdafx.cpp.md) | Anchor translation unit for the precompiled prelude; contributes nothing |

The record's declaration lives outside this directory, with the other cross-module
interfaces: [`Include/xrAPI/xrAPI.h`](../../Include/xrAPI/xrAPI.h.md).
