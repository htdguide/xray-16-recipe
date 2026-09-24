# src/xrGame/level_script.cpp

> The `level`, `game`, `relation_registry` and `actor_stats` script namespaces: the general-purpose surface a mod uses to ask about and manipulate the running world.

**Needs** — [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`PostprocessAnimator.h`](PostprocessAnimator.h.md) · [`actor_statistic_mgr.h`](actor_statistic_mgr.h.md) · [`raypick.h`](raypick.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`ui/UIGameTutorial.h`](ui/UIGameTutorial.h.md) · [`date_time.h`](date_time.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [`xrPhysics/PHCommander.h`](../xrPhysics/PHCommander.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data plus thin adapters

## Purpose

Most of a STALKER mod is written against this file. It is not the binding of one class — it
is four flat namespaces of free functions that reach into whatever part of the engine the
script needs: the weather, the clock, the map, the camera, the navigation mesh, the
faction relations, the tutorial sequencer, the physics scheduler. Almost every entry is a
one-line adapter over a subsystem documented elsewhere; what this page records is the
*surface* — which names exist, what each promises, and the handful of places where the
adapter itself makes a decision.

The names and signatures are frozen by conformance criterion 10. A rebuild may reorganize
everything behind them and may not rename any of them.

## State

```text
active_tutorial   : optional<Sequencer>   # the tutorial currently playing
previous_tutorial : optional<Sequencer>   # exactly one may be suspended beneath it
```

**Invariant** — tutorials nest exactly one deep. Starting a tutorial while one is running
pushes the running one aside and inherits its saved input receiver, so that when the nested
one ends, input returns to whatever had it before the *outer* one started. A third
simultaneous tutorial overwrites the suspended slot and loses that receiver; this is
asserted, not handled. A rebuild should use a stack and the constraint disappears.

**Invariant** — a tutorial started while the load screen is up is silently dropped. The
sequencer takes over input and rendering, and doing that during a level load leaves the
player looking at a frozen screen.

Everything else in the file is stateless adaptation.

## `CLevel::script_register`

**Contract** — the single registration entry point, called once at script-engine bring-up.
It declares three unnamespaced classes, four namespaces, and a set of enumerations. Listed
below by namespace; each group's entries are one-line adapters unless noted.

### Global classes and enumerations

- **`ray_pick`** — a reusable ray query object: set position, direction, range, target mask
  and an ignored object, then query and read back the result, the hit object, the distance
  and the hit bone. Exists alongside the one-shot `level.ray_pick` because a script that
  casts every frame should not rebuild the query.
- **`rq_result`** — one ray hit: object, range, and the hit element (the bone index, which is
  what makes located damage scriptable).
- **`rq_target`** — the ray target mask: nothing, dynamic objects, static geometry, shapes,
  obstacles, both, dynamic. Frozen values; they are bit flags in the collision database.
- **`game_difficulty`** — the four difficulty levels, by name.
- **`CEnvDescriptor`** — read-only fog density and far plane of the current weather state.
- **`CEnvironment`** — the weather system, with one method returning the current descriptor.

### Namespace `level` — the world

**Time and weather.** `get_weather` · `set_weather` · `set_weather_fx` ·
`start_weather_fx_from_time` · `is_wfx_playing` · `get_wfx_time` · `stop_weather_fx` ·
`environment` · `rain_factor`. Every setter is a **no-op in editor mode**: the editor owns
the environment and a script fighting it would make authoring impossible.

`get_time_days` · `get_time_hours` · `get_time_minutes` each decompose the game clock and
return one field. All three read the clock from the *level* when one is loaded and from the
alife simulation otherwise, which is what lets a script ask the time on a loading screen or
between levels. Splitting the full date three times to return one number each is wasteful and
a rebuild should expose the decomposition once.

`set_time_factor` sets the rate of game time against real time — server-side only, and
refused in editor mode. `get_time_factor` reads it from the level, not the server, so it
answers on a client too. `change_game_time` advances the clock by days, hours and minutes;
it advances *both* the alife simulation's clock and the environment's independently, which
is the file's one real ordering hazard — the two must be given the same delta or the sky and
the simulation disagree. Only meaningful in single player with an alife simulation running.

**Navigation.** `vertex_position` · `valid_vertex` · `vertex_id` · `vertex_in_direction` ·
`high_cover_in_direction` · `low_cover_in_direction`. The cover pair take a direction vector
and use only its heading — cover is a horizontal quantity, so pitch is discarded.
`vertex_in_direction` walks from a vertex as far as it can along a direction for at most a
given distance and returns where it stopped, **falling back to the starting vertex** rather
than an invalid one, so a script can always chain the result.

`vertex_id` carries a deliberate bug-for-bug compatibility hack: the invalid vertex
identifier is the largest 32-bit value, and the original binding converted it to a number
one larger. Scripts in the wild test against that larger number, so the export adds one to
exactly that value and returns everything else unchanged. Reproduce this. It is the clearest
case in the recipe of a frozen surface outliving its cause.

**Objects and spawning.** `object_by_id` returns the script facade for a live object by its
entity identifier, or nothing — the standard way a script gets a handle.
`iterate_online_objects` walks every possible identifier and calls a script function for each
live object, stopping when the function returns true; it is a linear scan of the entire
16-bit identifier space every call, and is the most expensive thing a script can do here.
`spawn_item` creates an entity from a section at a position, optionally under a parent — the
path for bolts, ammo and phantoms, which the alife simulation's own create call cannot make.
`spawn_phantom` is that call with a fixed section.

**Ray picking.** `ray_pick` casts from a point along a direction for a range against a target
mask, optionally ignoring one object, and reports whether it hit. `get_target_obj` ·
`get_target_dist` · `get_target_element` read the crosshair's *already-computed* query
instead of casting again — the head-up display casts that ray every frame regardless, so
asking for what is under the crosshair should be free.

**The map.** `map_add_object_spot` · `map_add_object_spot_ser` · `map_remove_object_spot` ·
`map_has_object_spot` · `map_change_spot_hint`. The `_ser` variant differs in exactly one
respect: the created location is marked *serializable* and therefore survives a save. A spot
added by the plain call is a runtime annotation and is gone on load. Adding an empty hint
leaves the location's own default hint in place rather than blanking it.

**Screen and input.** `start_stop_menu` toggles a dialog. `add_dialog_to_render` ·
`remove_dialog_to_render` · `main_input_receiver`. `hide_indicators` ·
`hide_indicators_safe` · `show_indicators` hide or show the head-up display and crosshair;
all three **also toggle a runtime invulnerability flag**, because hiding the interface means
a cutscene is playing and dying during one is not an outcome the game handles. The `_safe`
variant differs by not force-closing already-open dialogs and by notifying the screen that
the hiding came from outside, so that it can restore correctly.

`disable_input` · `enable_input` gate all player input. Disabling is **refused while the
actor is in god mode**, which is the debugging escape hatch from a script that disables input
and then fails before re-enabling it. `show_weapon` is refused the same way.

`show_minimap` · `hide_minimap` · `minimap_shown` · `get_fov` · `set_fov` ·
`get_active_cam` · `set_active_cam` · `get_actor_body_state` ·
`get_actor_body_state_wishful`. The camera setter validates the mode against the camera
count and silently ignores an invalid one; the getter answers a sentinel when the view is not
the actor.

**Camera and post-process effectors.** `add_cam_effector` · `add_cam_effector2` ·
`remove_cam_effector` · `add_pp_effector` · `remove_pp_effector` ·
`set_pp_effector_factor` · `add_complex_effector` · `remove_complex_effector`. The camera
effectors return the length of the animation they started, so a script can schedule what
happens when it ends without polling. The `2` variant additionally positions the camera
absolutely rather than relative to the actor and overrides the field of view. Removing a
post-process effector *fades it out over one second* rather than cutting it, which is why
removal is not symmetric with addition.

**Physics-scheduled script calls.** `add_call` · `remove_call` · `remove_calls_for_object` —
register a (condition, action) pair that the physics step evaluates every tick, calling the
action when the condition is true. Three overloads: two free functions; an object plus two
method *names*; an object plus two functions. The object-bearing forms exist so that the pair
can be removed later by identity, and so that a registration is dropped when its owning
script object goes away — the free-function form has no such hook and leaks its registration
until removed by hand. The name-based overload additionally registers *uniquely*, refusing a
duplicate pair; the others do not.

**Sounds.** `prefetch_sound` loads a sound before it is needed. `iterate_sounds` enumerates
a family of sound files by probing the virtual filesystem for a base name and then that name
with each integer suffix up to a limit, calling back for each one that exists. It is a
filesystem probe per candidate, and the limit is the script's, not the engine's — this is how
a script discovers how many variants of a bark were shipped without a manifest. Two overloads
differ only in whether the callback carries a bound object.

**Miscellaneous.** `name` (the loaded level's name) · `present` (whether any level is
loaded) · `game_id` · `get_start_time` · `get_bounding_volume` (the level's world-space
extent) · `patrol_path_exists` · `client_spawn_manager` · `get_snd_volume` ·
`set_snd_volume` (clamped to zero..one) · `send` (inject a network packet into the level's
stream, with the reliability flags exposed) · `set_game_difficulty` · `get_game_difficulty`.
Setting difficulty stores it globally *and* notifies the game mode, which is what re-applies
the per-difficulty modifiers; storing it without the notification changes nothing until the
next level load.

In a development build only: `debug_object`, `debug_actor` and `check_object`. The first two
log an error the first time they are called, because they resolve objects by *name* — a
linear scan and an unstable key — and exist only for the debugger's convenience.

### Namespace `game` — the clock, tutorials and text

- **`CTime`** — a first-class game timestamp with comparison and arithmetic operators, a
  seconds difference, component get and set, and formatting to date and time strings against
  the two format enumerations (date to day/month/year, time to hours/minutes/seconds/
  milliseconds). The binary operations guard against a missing operand and answer neutrally
  rather than failing, because a script passing nothing here is common and crashing the game
  over it is not proportionate.
- `time` · `get_game_time` — the current game time, as a number and as a timestamp.
- `start_tutorial` · `stop_tutorial` · `has_active_tutorial` · `active_tutorial_name` — see
  the nesting invariant above. Reading the name with no tutorial running is undefined.
- `translate_string` · `reload_language` — the string table lookup, and a reload for
  changing language at run time.
- `log_stack_trace` — dumps the native call stack; a diagnostic for a mod author chasing a
  crash.
- `jump_to_level` — three overloads, and they are two *different* operations. Given a level
  *name*, it asks the alife simulation to move the actor to that level, refusing with a
  logged error if no such level exists in the game graph — this is the safe, coarse form.
  Given a destination (position, navigation vertex, graph vertex and optionally angles), it
  sends the same level-change request a level changer sends, bypassing every check. The
  angle-less overload faces the actor along the world axes.

### Namespace `relation_registry` — faction standing

`community_goodwill` · `set_community_goodwill` · `change_community_goodwill` ·
`community_relation` · `set_community_relation` · `get_general_goodwill_between`. Communities
are named in script and resolved to indices here.

`get_general_goodwill_between` is the only one with an algorithm: the effective standing of
one character toward another is the **sum of three separate registrations** — the personal
goodwill between the two, the first one's community's goodwill toward the second
*individual*, and the relation between the two communities. That three-term sum is the whole
faction model and it is written down nowhere else. Both parties must be trader-capable alife
records; anything else logs an error and answers neutral.

### Namespace `actor_stats` — the end-of-game tally

`add_points` · `add_points_str` · `get_points`. Records scored events into a named section
with a detail key, either as a count-and-value pair or as a free string, and reads a
section's total. This is what fills the statistics screen.

### Unnamespaced

`command_line` · `IsGameTypeSingle` · `IsDynamicMusic` · `IsImportantSave` ·
`render_get_dx_level`. The last reports the running renderer's generation number, which is
how a script decides whether an effect it wants is available.

**Notes**

- Several `level` entries reach through a global actor reference with no check. A script
  calling them with no actor — a main menu, a dedicated server — crashes. The surface has
  always been this way and mods work around it by testing `level.present` first.
- The registration mixes two declaration styles: some classes are declared outside the
  module block and some inside. Only the inside ones actually reach the script namespace;
  the environment classes are declared outside and are reachable only through the
  `environment` function's return value. This is not deliberate design, but it is the shipped
  surface: the class names are not globally addressable from script.
