# src/xrEngine/EngineAPI.cpp

> Enumerates the renderers the build offers, picks one that this machine and this installation can actually run, and hands the game module its object factory.

**Needs** — [`EngineAPI.h`](EngineAPI.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [`xrScriptEngine`](../xrScriptEngine/README.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`EngineAPI.h`](EngineAPI.h.md); callers name that, not this file.
**Tier floor** — T1: it probes hardware through the graphics backends and queries the process's address-space limit.

## Purpose

Two module boundaries meet here. The renderer is *pluggable*: the build carries up to two renderer modules, each offering several named quality modes, and exactly one mode is live. The game is a module too: the engine never constructs a game entity itself, it calls a factory the game module installed, keyed by a class identifier read from level data.

The reason both are modules rather than direct calls is the same reason the engine has a global environment struct: the engine is below the game and below the renderer in the build order, and must drive both without naming either.

## State

```text
RECORD ModuleRegistry
  render_modes      : map<mode_name, renderer_module>   # every mode every module offers
  game_module       : optional<GameModule>
  selected_renderer : optional<RendererModule>
  create            : function(class_id) -> optional<object>
  destroy           : function(object)

# Published for the console and the options screen:
quality_choices : list<(mode_name, mode_index)>
```

The two factory functions start as stubs that create nothing and assert on any destroy, so a call before the game module is loaded fails loudly rather than crashing. That stub pair is the whole reason the factory is a pair of installable functions rather than a virtual call on the module: entity destruction happens during shutdown *after* the module has been finalized, and the stub must still answer.

## `build_renderer_list`

**Contract** — Asks each renderer module which quality modes it supports on this machine, and merges them into one name-keyed table plus an ordered choice list for the options screen. Probing is slow — it creates and destroys real graphics devices to test capabilities — so the result is cached and a second call returns immediately. A duplicate mode name is skipped rather than overwriting, so the first module to claim a name keeps it. Fails hard only for the dedicated server, which requires the first module.

```text
FUNCTION build_renderer_list(modules)
  IF quality_choices is not empty THEN RETURN      # already probed

  FUNCTION load(module) -> bool
    IF module is absent THEN RETURN false
    modes = module.probe_supported_modes()         # performs hardware tests; slow
    IF modes is empty THEN RETURN false
    FOR EACH (name, index) IN modes
      IF name already claimed THEN skip
      render_modes[name] = module
      quality_choices.append(name, index)
    RETURN true

  IF dedicated_server THEN
    FAIL WITH "dedicated server needs the base renderer" IF NOT load(modules[0])
  ELSE
    load each module in turn, ignoring failures
  log the resulting mode list
```

**Notes** — A module reporting an empty mode list is a normal outcome, not an error: it means "this backend cannot run on this machine at all". Only the dedicated server treats it as fatal, and only for the base module, because the server needs a renderer object to exist even though it draws nothing.

**Notes** — The mode index a module reports alongside each name is the quality tier the *game* uses to gate content — which detail meshes load, whether grass exists. It is not the renderer's internal identifier, and it is why the choice list carries a number as well as a name.

## `select_renderer`

**Contract** — Resolves the console's `renderer` variable to a live renderer module. Tries the requested mode first; if it is unknown, or its module reports that this installation does not meet its requirements, falls through to the first mode in the table that does and writes that choice back into the console so the settings file records it. Fails hard when nothing is usable. Finishes by asking the winning module to fill the global environment struct with its own interfaces.

```text
FUNCTION select_renderer()
  requested = console.get("renderer")
  IF render_modes has requested AND its module.requirements_met() THEN
    selected = that module
  IF selected is absent THEN
    FOR EACH (name, module) IN render_modes
      IF module.requirements_met() THEN
        selected = module ; requested = name
        console.run("renderer <name>")     # persist the downgrade
        BREAK
  FAIL WITH "can't setup renderer" IF selected is absent
  selected.install_into_global_environment(requested)
```

**Invariants** — After this returns, the global environment struct holds a live renderer, render factory and everything else the module publishes. Every subsequent graphics call in the engine reaches the backend through it.

**Notes** — "Requirements met" is two questions in one: does the hardware support this generation, and does this *installation* contain the shader sources this generation needs. The second matters because shaders ship with the game data and a Shadow of Chernobyl installation does not contain the newest generation's shaders. A silent downgrade is the right answer to both.

**Notes** — The fallback iterates the mode table in its own order, which is name-keyed and therefore alphabetical rather than best-first. The downgrade target is thus not necessarily the *next best* renderer. A rebuild should order the fallback by quality.

## `initialize`

**Contract** — Selects the renderer, then hands the game module the two factory function slots to fill. Asserts that the module filled both. After this, entity creation by class identifier works.

## `destroy`

**Contract** — Finalizes the game module, drops the selected renderer, resets the factory slots, and compacts the shared collision-query scratch pool.

**Notes** — Compacting the collision scratch pool here is placement by convenience rather than design — it is a per-thread result buffer that grew to the largest query of the session and is simply the last big thing to release. The original notes it lives here because of a link-order problem, not a design one.

## `FactoryObject`

**Contract** — What the engine demands of every object the game's factory produces: it reports its own class identifier, and it can return itself as the base interface. The class identifier is the key read from the level's spawn file, so it must round-trip exactly.

**Notes** — The self-returning construct step exists to let a derived class that inherits the base more than once resolve to a single instance. That is a C++ inheritance artifact with no meaning in a rebuild; the decision it encodes is just "an object knows its own identity".

## `GameModule`

**Contract** — What the engine demands of the game: install and uninstall the object factory, and create and destroy the persistent game object — the one long-lived game-side object that outlives every level. Four entry points, and they are the *entire* surface through which a 200,000-line game is driven.

## `RendererModule`

**Contract** — What the engine demands of a renderer: report the quality modes it supports on this machine, report whether its requirements are met, install its interfaces into the global environment for a named mode, and remove them. Probing is allowed to be expensive and is called once.

## `is_enough_address_space_available`

**Contract** — Reports whether the process can address enough memory to run the high-quality settings. On a 32-bit-limited host it asks the operating system for the highest usable application address and compares against a threshold; everywhere else it reports true unconditionally.

**Notes** — The threshold is the point above which a 32-bit process has the large-address-aware flag and therefore roughly three gigabytes rather than two. The check is exported *to the shipped Lua scripts*, which call it to decide whether to offer the highest texture-detail setting. That makes both the name and the boolean it returns frozen by the script seam, even though on a 64-bit build the answer is always the same.

**Notes** — A second function is exported to Lua alongside it, asking whether the hardware supports the second-generation renderer, and it now returns true unconditionally. It exists because shipped scripts call it; the honest answer on any machine that can run this build is yes.
