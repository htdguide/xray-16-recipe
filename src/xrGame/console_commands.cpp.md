# src/xrGame/console_commands.cpp

> Declares the game layer's entire console surface: every named runtime setting and every command a player, modder or tester can type.

**Needs** — [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`Actor_Flags.h`](Actor_Flags.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_debug.h`](ai_debug.h.md) · [`ai_debug_variables.h`](ai_debug_variables.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_single.h`](game_cl_single.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`saved_game_wrapper.h`](saved_game_wrapper.h.md) · [`autosave_manager.h`](autosave_manager.h.md) · [`character_hit_animations_params.h`](character_hit_animations_params.h.md) · [`attachable_item.h`](attachable_item.h.md) · [`attachment_owner.h`](attachment_owner.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`mt_config.h`](mt_config.h.md) · [`date_time.h`](date_time.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrEngine/xr_ioc_cmd.h`](../xrEngine/xr_ioc_cmd.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrPhysics/console_vars.h`](../xrPhysics/console_vars.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md) · [`xrScriptEngine/script_profiler.hpp`](../xrScriptEngine/script_profiler.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`console_commands_mp.cpp`](console_commands_mp.cpp.md)
**Tier floor** — T2: text parsing and pointers to live tuning variables; nothing here touches a device or a format

## Purpose

The console is the engine's control surface, and its **names are frozen**. Shipped
configuration files set console variables by name, the options screen writes them by name,
user settings files persist them by name, and a decade of mod documentation names them. A
rebuild that renames a variable breaks data it did not write. Conformance criterion 10
covers the script surface; this file is the same obligation for the console.

Two things happen here. Most of the file defines *command types* — a handful of general ones
(a bounded integer, a bounded float, a bit of a flag word, a named token, a vector) plus
about sixty special ones that do something when set rather than merely holding a value. The
last function then **registers every name**, binding it either to a general type pointed at
a live variable or to one of the special ones.

The registration function is the authoritative list of the surface, and it is worth reading
as data rather than as code.

## State

`Stateless` as a module. It holds references to tuning variables owned elsewhere, plus two
values of its own: the name of the most recently saved game (so that "load the last save"
has something to load, and so that it can be written into the user's settings file), and a
flag word of miscellaneous game-wide options.

## build tiers

**Contract** — the surface is not one set. Three tiers, decided at build time:

```text
always present            the player-facing settings: interface, controls, difficulty,
                          language, save and load, the field of view, the heads-up display
not in the shipping build the cheats and the developer conveniences: god mode, no-clip,
                          spawn, run a script, set the time factor, jump to a level,
                          the alife tuning knobs, the multithreading switches
debug builds only         the drawing overlays, the dumps, the tuning of things a player
                          never sees -- physics debug rendering, animation tracking,
                          navigation-graph visualization
```

**Invariants** — the boundary between the first two tiers is a game-design decision, not a
technical one: a command that lets a player spawn a weapon or turn off damage must not exist
in a shipped multiplayer-capable build. A rebuild must keep the gate, and must keep it at
*registration* — an unregistered name is not merely refused, it does not autocomplete and
does not appear in the listing, which is what stops a player discovering it.

## the general command types

**Contract** — five shapes cover most of the surface, and each is a name bound to the address
of a live variable:

```text
bounded integer   name, variable, minimum, maximum
bounded real      name, variable, minimum, maximum
flag bit          name, flag word, bit mask      -- "on"/"off"/"1"/"0"
named token       name, variable, a table of (text -> value)
bounded vector    name, variable, component-wise minimum and maximum
bounded text      name, buffer, capacity
```

A command also answers three questions besides "execute": its current value as text, a
one-line description, and a list of completion suggestions. The suggestion list is what makes
the console usable — for a save name it lists the save files, for a level name the levels in
the game graph, for a script the script files — and it is a real part of the contract, not
decoration.

**Invariants** — a bounded command clamps rather than rejects. A value outside the range is
brought inside it, because these are read from a user's settings file written by an older
build whose ranges differed.

**Notes** — commands persist themselves into the user's settings file, each writing its own
line. A command that should not persist — a one-shot action, a debug toggle — overrides that
to write nothing. Which commands persist is therefore part of the surface too.

## a mutually exclusive pair

**Contract** — two flag bits that must never both be set are registered as a *radio group*:
setting either one clears the other. The shipped instance is the pair that force a creature
to always, or never, use the physics-driven movement path — settings that contradict each
other.

```text
FUNCTION radio_execute(which, args)
  set `which` from args as an ordinary flag bit
  IF which is now set THEN clear BOTH bits, then set `which`
```

**Notes** — the implementation clears both and re-sets the chosen one, which is the correct
order; clearing the other first and then setting would leave a window where neither is set
if the two aliased.

## the commands that do something

The general types cover settings. These are the entries whose execution has consequences,
and they are what a rebuild must reproduce behaviourally rather than by name alone.

### `save`, `load`, `load_last_save`

**Contract** — saving and loading are **not** performed here. Each validates and then sends a
message to the authoritative side, which performs the operation at a point in the frame where
the world is quiescent. That is the whole reason these are not direct calls: a save taken
mid-frame from the console captures a half-updated world.

```text
FUNCTION save(name)
  IF not a single-player game THEN refuse
  IF the actor is absent or dead THEN refuse     # a save you cannot load back to
  IF name is empty THEN
    name = "<user> - quicksave"; flag = 0        # flag 0: the implicit quicksave slot
  ELSE
    IF name contains any of  / \ : * ? " < > | ^ ( ) [ ] %  THEN refuse
    flag = 1
  send save_game(name, flag) to the authoritative side
  show the "game saved" notice, naming the file
  take a thumbnail screenshot into $game_saves$/<name>.dds
```

```text
FUNCTION load(name)
  IF the alife simulation is not running THEN refuse
  IF name is empty THEN refuse
  IF no such save exists THEN refuse
  IF the save's version does not match, or it is corrupt THEN refuse
  IF the name is not a valid file name THEN refuse
  close the main menu; unpause
  send load_game(name) to the authoritative side
```

**Invariants** — the file-name validation exists because the name is typed by a human and
becomes a path. Rejecting the separator characters, the wildcards and the shell
metacharacters is the whole defence, and it is applied on **both** save and load.

**Invariants** — the version check on load is what turns "an old save silently produces a
broken world" into a refusal. A rebuild must keep a version stamp in the save and check it
here.

**Notes** — the name of the last save is remembered in a variable that is itself persisted
into the user's settings, so "load the last save" survives a restart. Calling that command
*with* an argument does not load — it sets the remembered name. That overloading is
surprising and a rebuild should split it.

**Notes** — the save file's extension differs between the newest game and the two older ones,
and the completion list picks between them by which game's data is mounted. The save format
is the same; only the extension differs.

**Notes** — both commands ask the memory-statistics command to run first. That is
instrumentation left switched on: saves and loads are where memory problems surface, and the
log line before each is worth having.

**Notes** — the readiness protocol from
[`autosave_manager.cpp`](autosave_manager.cpp.md) is present in the save path but
**disabled**. A manual save is taken regardless of whether a subsystem has declared itself
mid-transaction.

### `g_game_difficulty`

**Contract** — a named token, and after setting it, the single-player game is told the
difficulty changed so it can retune what depends on it. Refuses outside single player, where
the setting is meaningless.

### `g_language`

**Contract** — a named token over the languages the string table offers. After setting it,
the string table reloads, the interface is rebuilt, and **every inventory item in the world
is told to re-read its display name**. Item names are resolved from the string table at
spawn and cached, so without that sweep the language changes everywhere except in the
player's backpack.

**Invariants** — the sweep walks the entity identifier space exhaustively rather than a
registry. A rebuild with an object registry should iterate that instead; what matters is that
*every* live item is visited.

**Notes** — the token table is fetched lazily from the string table and, if it is missing, the
string table is torn down and reinitialized to produce one. That is a guard against
initialization order, not behaviour.

### the alife tuning knobs

**Contract** — five settings that change how the off-screen simulation runs, all refusing
outside single player with an alife simulation:

- **the game time factor** — how fast game time runs relative to real time. Applied only on
  the authoritative side.
- **the switch distance** — how close an offline entity must come to the player before it is
  promoted online. Sent as a *message*, not set directly, because it is authoritative state
  that clients must agree on. Refuses values below two metres, which would make promotion
  effectively never happen.
- **the process time** — the millisecond budget the alife update may consume per turn.
  Refuses values below one.
- **the objects per update** — how many offline entities are advanced per turn.
- **the switch factor** — clamped to between a tenth and one; scales the switch distance for
  demotion, so that an entity leaving does not immediately re-enter. Hysteresis.

**Invariants** — the switch factor's clamp *is* the hysteresis guarantee: a factor of one
means promotion and demotion happen at the same distance and an entity standing on the
boundary flickers between online and offline every update. A rebuild must keep the upper
clamp.

### `ph_frequency` and `ph_iterations`

**Contract** — the rigid-body solver's step rate and solver iteration count. The rate is
entered as a *frequency* and stored as a step duration, and setting it retunes the live
world. The allowed range is much narrower in a shipping build — fifty to two hundred steps
per second — than in a debug one, because a physics rate outside that range makes the shipped
content behave wrongly rather than merely slowly.

### `time_factor`, `time_factor_single`, `start_time_single`

**Contract** — the real-time dilation of the whole simulation, clamped to a thousand times;
the game-clock factor; and the in-game date and time the next game starts at. Developer
commands, absent from a shipping build.

### `ui_time_factor`, `time_dilation_inventory`, `time_dilation_pda`

**Contract** — the newest game's "time keeps running while you are in a menu" feature: a
factor at most one applied while an interface screen is open, and a switch per screen kind
saying whether that screen dilates time at all. Player-facing and always present.

### `g_spawn`, `g_spawn_to_inventory`

**Contract** — create an entity of a named configuration section at the player's position, or
directly into the player's inventory. Both refuse outside single player and both check the
section exists first, reporting the name if it does not — because the overwhelmingly common
use is a typo.

**Notes** — the world spawn goes through the *local* spawn path with a wildcard team; the
inventory spawn goes through the ordinary item-spawn path with the actor as parent. They are
different code paths, not two arguments to one.

### `jump_to_level`

**Contract** — move the player to another level through the alife simulation, which is what
makes it a legal transition rather than a teleport: the entity is demoted, moved on the game
graph, and re-promoted. Validates the name against the game graph's level list and offers
that list as completions.

### `run_script`, `run_string`

**Contract** — run a script file, or evaluate a line of script. Both rescan the script path
first so a file edited while the game runs is picked up. `run_string` preserves the argument's
letter case, unlike every other command — script identifiers are case-sensitive and the
console lowercases arguments by default.

**Invariants** — when the level's script process exists, both queue into it so the script runs
at the script layer's own point in the frame. `run_string` falls back to evaluating
immediately when there is no level, which is how a script can be run from the main menu.

### `g_god`, `g_no_clip`, `g_unlimitedammo`

**Contract** — the cheats, absent from a shipping build. Turning off collision additionally
nudges the actor's physics once, because the actor has no physics representation at all until
it first moves and the collision state would otherwise not take effect until then.

### `ui_style`, `ui_restart`

**Contract** — select an interface skin by name from those the skin loader found, applying it
immediately; and reload the current one, which is how a skin author iterates without
restarting.

### `lua_gc_method` and the collector knobs

**Contract** — chooses among four script garbage-collection strategies: off, incremental
steps, a periodic timeout, and one full collection. Selecting the full collection performs it
and then **reverts to the previous strategy**, so it behaves as a one-shot action despite
being registered as a setting.

**Invariants** — switching away from "off" must restart the collector, and switching to "off"
must stop it. Setting the strategy variable without telling the script machine leaves the two
disagreeing.

### the script profiler

**Contract** — eight separate command names sharing one implementation, which dispatches on
its own registered name: report status, start in the default mode, start in hook mode, start
in sampling mode (with an optional interval), stop, reset, print, and save to a file.

**Notes** — dispatching on the command's own name rather than on an argument is what gives
each mode its own completion entry and its own help line. A rebuild with subcommands gets the
same effect.

### `dbg_adjust_attachable_item`

**Contract** — selects an item on the player, by section name, as the target of the live
attachment-offset nudging described in
[`attachable_item.cpp`](attachable_item.cpp.md); calling it again deselects. Looks first
among the player's attached items and then at the active item, so both a belt artefact and a
held weapon can be adjusted.

### `dbg_var`

**Contract** — reads a named debug variable with one argument, writes it with two. The
variable set is the AI debug system's own, addressed by name at run time, which is how a
developer exposes a new tuning value without adding a console command for it.

### the dumps and the overlays

**Contract** — debug-only: dump the player's known information, the task list, the map, every
creature, every object, every bone of a model; clear the log; force a crash to test the
crash handler; and roughly a hundred drawing toggles grouped into flag words — the AI system's,
the physics system's, the networking system's, and the animation tracker's.

**Notes** — these are grouped into *flag words* rather than separate variables so that a whole
family can be saved, restored and cleared as one value, and so that hot code can test one word
instead of many booleans. That is worth preserving: the alternative costs a branch per
overlay in paths that run per object per frame.

### `slot_0` … `slot_3`

**Contract** — four persisted strings naming the item sections bound to the quick-use keys.
Player settings, stored by name so they survive a save and a game.

### `stat_memory`

**Contract** — prints the memory breakdown — renderer textures, process heap, the shared
string pool's saving, the shared-memory pool's saving — and **installs itself as the
out-of-memory handler**, so the same breakdown is printed automatically when an allocation
fails. That registration is the useful part: the report a crash needs is the report a
developer asks for.

## `CCC_RegisterCommands`

**Contract** — called once during bring-up, before any settings file is read. Registers every
name in the surface and sets the initial state of the flag words whose defaults are not zero:
the heads-up display starts with the crosshair, the weapon, the frame and the information
panel visible; the multithreading word starts with every eligible subsystem enabled; the
common flags start with dynamic torch lighting on.

It ends by calling the multiplayer registration in
[`console_commands_mp.cpp`](console_commands_mp.cpp.md), which is a separate file only because
of its size.

**Invariants** — defaults are set **here**, before the user's settings file is applied, so that
the file overrides them rather than the reverse. A rebuild that initializes these at their
point of declaration must still guarantee that ordering.
