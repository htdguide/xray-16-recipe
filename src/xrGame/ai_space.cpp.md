# src/xrGame/ai_space.cpp

> The AI layer's single global: it owns the navigation graphs, the cover database, the dynamic-obstacle registry, the door manager, the evaluation-function store and the script virtual machine, and it defines the order in which all of them come up and go down around a level load.

**Needs** — [`ai_space.h`](ai_space.h.md) · [`ai_space_inline.h`](ai_space_inline.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_point.h`](cover_point.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`doors_manager.h`](doors_manager.h.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`xrAICore/AISpaceBase.hpp`](../xrAICore/AISpaceBase.hpp.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/graph_engine.h`](../xrAICore/Navigation/graph_engine.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`ai_space.h`](ai_space.h.md)
**Tier floor** — T3: lifecycle orchestration; nothing here touches a byte layout.

## Purpose

Every AI query in the game — "which vertex is this position", "where is cover from that
direction", "what does the off-screen simulation say about this entity", "call this Lua
function" — reaches its answer through one process-global object. This file is that
object: its construction, its level-load and level-unload sequences, and the script
engine's startup.

It is a global rather than an injected dependency because it is reached from thousands of
places including deeply nested geometry code; §7 of the system requirements calls this
pattern out as the engine's deliberate cycle-breaker, and a rebuild should make the same
set of services explicit dependencies of whatever needs them.

## State

```text
RECORD AiSpace
  inited          : bool
  events          : notifier with two events
                       ScriptEngineStarted, ScriptEngineReset
  ef_storage      : evaluation-function store    # the tuned scoring functions creatures
                                                 #   rank targets and positions with
  cover_manager   : cover database               # static cover, computed at level load
  moving_objects  : dynamic obstacle registry
  doors_manager   : door graph for the loaded level
  alife_simulator : optional reference to the off-screen simulation (not owned)
  # inherited from the AI-core base: the game graph, the level graph, the cross table
  # between them, the path-search engine and the patrol path storage

# Invariant: init runs exactly once per process.
# Invariant: the alife simulator reference toggles — it is only ever set from absent
#   to present or present to absent, never replaced. See set_alife.
# Invariant: on a dedicated server, none of the owned services exist. The server runs
#   the authoritative simulation without navigation, cover, scripts or doors.
```

## `GetInstance`

**Contract** — Returns the process-global AI space, creating and initializing it on first
call. Not thread-safe: first touch must happen on the main thread during startup.

**Notes** — The short spelling `ai()` is the name used everywhere else and is the same
call. That abbreviation is why the global is tolerable in the source; a rebuild loses
nothing by giving the services proper names.

## `init`

**Contract** — Brings the AI space up. Asserts it has not run before.

```text
FUNCTION init()
  IF this process is a dedicated server
    mark inited and RETURN     # no navigation, no cover, no scripts, no doors
  bring up the AI-core base (graphs, path engine, patrol paths)
  create the evaluation-function store
  create the cover manager
  create the dynamic obstacle registry
  create the script virtual machine      # asserted absent beforehand
  RestartScriptEngine()
  inited = true
```

**Invariants** — The script engine is created here and nowhere else; the assertion that it
does not already exist makes the AI space its sole owner. The ordering matters in one
direction only: the script engine's startup registers classes through the object factory,
which must already exist as a global.

## `SetupScriptEngine` and `RestartScriptEngine`

**Contract** — `SetupScriptEngine` initializes the virtual machine, exports every engine
class to it, lets the script side register its own classes with the spawn factory, and
loads the common script files. `RestartScriptEngine` wraps that between two notifications
so that every subsystem holding a script reference can drop it and re-acquire it.

```text
FUNCTION RestartScriptEngine()
  IF a script engine exists
    fire ScriptEngineReset        # subscribers must release every script handle NOW
  SetupScriptEngine()
  clear the dynamic obstacle registry     # checked builds only
  IF a script engine exists
    fire ScriptEngineStarted      # subscribers may re-acquire

FUNCTION SetupScriptEngine()
  initialize the virtual machine and export the full engine class surface
  RegisterScriptClasses()       # script-defined classes into the spawn factory
  register the spawn factory itself into the script namespace
  LoadCommonScripts()
  IF not a shipping gold build
    have the spawn factory re-read its spawn data
```

**Invariants** — The reset event must fire *before* the machine is torn down and the
started event *after* it is up: a subscriber that holds a Lua reference across the gap
holds a dangling one. This is the game-side half of the requirement in §6 that the script
stack stay balanced across every boundary.

**Notes** — The obstacle registry is cleared only in checked builds. That asymmetry is
suspicious: if clearing it is correct after a script restart, it is correct in every
build; if it is not, it is a debugging aid whose presence changes behaviour. A rebuild
should decide.

## `RegisterScriptClasses`

**Contract** — Reads the script configuration file, takes the comma-separated list of
*class registrator* function names from its common section, and calls each one with the
spawn factory. A missing function is logged and skipped, not fatal. A configuration file
without a common section means no script classes, silently.

**Notes** — This is the mechanism that lets a mod add entity classes: the game's class
identifier table is not closed at compile time, and the registrators run before any spawn
record is read. The file name and the key are frozen by the shipped data.

## `LoadCommonScripts`

**Contract** — From the same configuration file's common section, loads each named script
file into the global namespace. Same forgiving shape: absent section or absent key means
nothing to load.

**Notes** — Both routines open, parse and discard the same configuration file separately.
That is wasteful and harmless; a rebuild reads it once.

## `load`

**Contract** — Brings the AI space up for one named level. Requires the game graph to be
present already — the cross-level graph outlives individual levels and is loaded by the
off-screen simulation.

```text
FUNCTION load(level_name)
  REQUIRE the game graph exists
  unload(reload = true)                  # tear the previous level down first
  load the level graph and cross table for this level
  cover_manager.compute_static_cover()   # derive cover values over the new level graph
  moving_objects.on_level_load()
  doors_manager = new door manager over the level graph's bounding box
```

**Invariants** — The order is load-bearing end to end: static cover is derived *from* the
level graph so it cannot precede it, and the door manager is constructed with the level's
spatial extent so it cannot precede it either. The unload-first step means loading a level
while one is loaded is well defined rather than an error.

## `unload`

**Contract** — Tears down everything level-scoped: the script engine's per-level state, the
door manager, then the AI-core base's graphs. A dedicated server returns immediately, since
it never built any of it. The *reload* flag is passed down to the base so it can keep what
survives a level change.

**Invariants** — Scripts go first. A script holding a reference to a navigation vertex or a
door must be given the chance to drop it before the structure behind it is freed.

## `set_alife`

**Contract** — Attaches or detaches the off-screen simulation. Asserts the transition is a
toggle — never a replacement — and that attaching happens while no game graph is loaded.
Detaching also releases the game graph, because the cross-level graph belongs to the
off-screen simulation's lifetime, not to the AI space's.

**Invariants** — The two assertions together encode the rule: the off-screen simulation
owns the game graph's lifetime, and the AI space merely borrows it for the duration.

## `Subscribe` / `Unsubscribe`

**Contract** — Register and cancel a callback on one of the two script-engine lifecycle
events. Registration returns a handle used to cancel. Every subsystem that caches anything
derived from the script state is expected to subscribe.

## Destruction

**Contract** — Fires the reset event if a script engine exists, unloads, then destroys the
script engine. The original wraps that final destruction in a catch-everything because a
Lua teardown can throw from a finalizer; a rebuild must decide what a failing script
finalizer means at process exit — the engine's answer is "ignore it", which is defensible
only because the process is ending.
