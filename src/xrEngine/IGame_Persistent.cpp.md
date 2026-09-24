# src/xrEngine/IGame_Persistent.cpp

> The layer that outlives any one level: it holds the weather, the particle world, the spatial databases and the level catalogue, and it drives the start / load / disconnect lifecycle.

**Needs** — [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md) · [`Environment.h`](Environment.h.md) · [`ILoadingScreen.h`](ILoadingScreen.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [`PS_instance.h`](PS_instance.h.md) · [`StringTable/StringTable.h`](StringTable/StringTable.h.md) · [`GameFont.h`](GameFont.h.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`IGame_Persistent.h`](IGame_Persistent.h.md)
**Tier floor** — T2: catalogue, lifecycle and lists; the only T1 pressure is that the loading screen draws from inside a nested render bracket.

## Purpose

The engine has two lifetimes: the process, and the level. This file is the thing that lives
at the first scale and owns everything the second scale needs handed to it. When a level
is torn down and another is built, what survives is here: the weather system, the particle
world, the two spatial databases, the object pool, the catalogue of installed levels, the
loading screen, and the sound scene.

It is also the **lifecycle driver**. Starting a game, loading one, disconnecting from one
and playing back a recorded match are four deferred events, and this file is their handler.
The game module subclasses it — `CreateLevel` and `DestroyLevel` are the two places where
the engine asks the game for a concrete level and hands it back — which is one half of the
engine ⇄ game interface described in the chapter README.

The abstraction is not clean: the class mixes a service locator (here is the weather, here
is the loading screen, here is the main menu), a lifecycle state machine, and a particle
garbage collector. A rebuild should split those three; they share nothing but their
lifetime.

## State

```text
RECORD Persistent
  game_params         : GameParams     # what the current session is
  environment         : Environment    # the weather and time-of-day system
  object_pool         : ObjectPool
  spatial_objects     : SpatialDB      # the world's renderable/queryable objects
  spatial_physics     : SpatialDB      # the physics collision broadphase
  loading_screen      : LoadingScreen
  main_menu           : optional<MainMenu>
  sound_scene         : SoundScene     # the process-wide default scene

  levels              : list<LevelInfo>  # every installed level, by folder
  current_level       : int              # index into levels, or none

  particles_active    : set<ParticleInstance>
  particles_to_play   : list<ParticleInstance>
  particles_to_destroy: list<ParticleInstance>

  load_depth          : int            # reference count; see LoadBegin
  load_stage          : int
  max_load_stage      : int
  phase_timer         : Timer
  loaded              : bool

RECORD LevelInfo
  folder : text       # with a trailing separator; this IS the level's identity
  name   : text       # display name; unused by this file — always absent

RECORD GameParams                      # parsed from one '/'-separated string
  game_or_spawn : text                 # a level name, or a spawn file name
  game_type     : text                 # "single", "deathmatch", ...
  alife         : text                 # "alife" enables the off-screen simulation
  new_or_load   : text
```

**Invariants**

- `load_depth` is zero exactly when no load is in progress, and `loaded` is true exactly
  then. Loads nest — a level load that triggers a script load that triggers another — so
  the timing and the "done" flag are keyed off the count reaching zero, not off any single
  call returning.
- A level exists in the catalogue only if all four of its required files exist; see
  `Level_Scan`.
- `particles_to_destroy` is empty across a game-type change; asserted.

**Notes** — `GameParams` is four fixed-size text fields that are *also* addressable as an
array of four, because the parse fills them positionally from one slash-separated string.
That overlap is a C++ union trick; the decision under it is that the session is described
by one string of the form `level/type/alife/mode`, positional, lowercased, which appears on
the command line, in the console `start` command, and in a demo file's header. A rebuild
parses it into a record with four named fields.

## The four lifecycle events

Everything that changes the session arrives as a deferred event (see
[`EventAPI.cpp`](EventAPI.cpp.md)), never as a direct call, because every one of them
destroys or creates the level from underneath whatever asked.

```text
FUNCTION on_event(persistent, event, p1, p2)
  CASE event IS "start"                      # p1 = server options, p2 = client options
    current_level = none
    REQUIRE no level exists
    close the main menu; hide the console
    pre_start(server_options)                # may end the current game type
    level = game.create_level()              # the game module supplies the concrete level
    REQUIRE level EXISTS
    load_begin()
    start(server_options)                    # may begin a new game type
    level.net_start(server_options, client_options)
    load_end()
    release both option strings              # they were handed over as raw ownership

  CASE event IS "disconnect"
    IF a quit is already queued
      release the input grab                 # the window is about to go
    IF a level exists
      remember whether the console was visible
      hide the console
      level.net_stop()
      game.destroy_level(level)
      restore the console's visibility
      IF neither a quit nor another start is queued
        cycle the main menu off and on again  # see note
    disconnect()                             # subclass hook; drops pending particles

  CASE event IS "start_mp_demo"              # p1 = demo file name
    REQUIRE no level exists
    close the main menu; hide the console
    reset the graphics device
    level = game.create_level()
    server_options = level.open_demo_file(demo)
    pre_start(server_options)
    load_begin()
    start("")                                # deliberately empty: see note
    level.net_start_play_demo()
    load_end()
```

**Notes** — four decisions here, none obvious.

**The main menu is cycled off and then on** after a disconnect, rather than simply shown.
Turning it off first forces it through its own teardown, which is what releases the screens
that were built against the level that has just gone. Showing it directly would leave
widgets holding dead references.

**Starting a demo passes empty options to `start`** while passing the real ones to
`pre_start`. The two differ in exactly one respect: `start` is what triggers the object
prefetch on a game-type change. A demo plays back a match whose assets are already
resolved by the recording, so paying the prefetch again is wasted. The source says so in
three words; the reason is that.

**A demo resets the graphics device first.** Demo playback runs at whatever resolution the
demo was recorded for, and the reset is how that is applied before anything is loaded.

**The option strings are transferred by ownership through the event queue.** The event
queue carries two integer-sized payloads and nothing else, so a string is handed over as a
pointer the receiver must release. That is a constraint of the queue, not a decision: a
rebuild with a typed event payload deletes this entirely.

## `PreStart` / `Start` — the game-type transition

**Contract** — between them they answer one question: did the *game type* change? Almost
everything expensive hangs off that answer, because a game type determines which objects
are prefetched and which rules the game module installs.

```text
FUNCTION pre_start(persistent, options)
  parse options INTO a scratch copy of the parameters
  IF the scratch game type differs from the current one
    on_game_end()                # release the previous type's prefetched objects and models

FUNCTION start(persistent, options)
  previous = current game type
  parse options INTO the live parameters
  IF the live game type differs from previous
    IF it is non-empty
      on_game_start()            # prefetch the new type's objects
  ELSE
    update_game_type()           # same type, new session: the subclass re-reads its rules
  REQUIRE no particles are pending destruction
```

**Notes** — the split into two calls is what makes the ordering safe: the *old* type's
resources must be released before the level is created, and the *new* type's prefetch must
happen after. The level creation sits between them.

`on_game_start` raises the "prefetching objects" title on the loading screen and then runs
the prefetch, unless the command line suppressed it. Suppressing it is a development
convenience that trades the first minute of stutter for a faster start.

## `Level_Scan` and `Level_Append` — the level catalogue

**Contract** — enumerates the top-level folders under the levels root and admits each one
that looks like a level. Rebuilt from scratch on every call, since an archive mount can add
levels at run time. Logs and gives up if the root has no folders at all.

```text
FUNCTION level_append(persistent, folder)
  IF ALL OF (folder+"level", folder+"level.ltx",
             folder+"level.geom", folder+"level.cform") exist under the levels root
    append LevelInfo(folder) TO levels

FUNCTION level_scan(persistent)
  release and clear levels
  folders = list the immediate subfolders of the levels root
  IF none
    log "no levels found" AND RETURN
  FOR EACH folder IN folders
    level_append(folder)
```

**Notes** — the four-file test *is* the definition of an installed level, and each file is
a different necessity: the chunked geometry container, the level's own configuration, the
render geometry and the collision model. A folder missing any one of them is an incomplete
install or a leftover, and admitting it would fail later with a worse message.

The folder string, trailing separator included, is the level's identity everywhere in the
engine. `LevelInfo` also carries a display name that nothing in this file ever sets; it is
always absent and a rebuild should drop it.

## `Level_Set`

**Contract** — makes one catalogued level current: repoints the `$level$` logical path at
its folder, records the index, and chooses a loading-screen image for it. Out-of-range
indices are ignored silently.

```text
FUNCTION level_set(persistent, index)
  IF index IS out of range
    RETURN
  point the "$level$" logical root AT levels[index].folder
  current_level = index

  # count the numbered intro images this level ships
  count = 0
  WHILE an image exists at "intro/intro_<folder>_<count+1>"
    count = count + 1

  IF count > 0
    pick a uniformly random one of them
  ELSE IF an unnumbered "intro/intro_<folder>" exists
    use it
  ELSE IF a "intro/intro_no_start_picture" exists
    use it
  ELSE
    use nothing

  IF an image was chosen
    loading_screen.set_level_logo(it)
```

**Notes** — the numbered images are searched for consecutively from one and stop at the
first gap, so a level shipping images 1, 2 and 4 offers only the first two. That is a
convention the shipped data follows and a rebuild must keep, because it is how the game
data expresses "pick a random loading image for this level".

Images are looked for in *two* logical roots — the shared texture root and the level's own
— which is what lets a modification ship a loading screen beside its level without touching
the base game's texture archive.

**Retargeting `$level$` is the load-bearing act here.** Every subsequent file the loader
opens is addressed relative to that root, so setting the current level is literally
re-pointing one entry in the virtual filesystem's root table.

## `Level_ID`

**Contract** — resolves a level name and version to a catalogue index, mounting the archive
that carries it if it is not yet mounted. Optionally makes it current. Returns none when
the name is unknown.

```text
FUNCTION level_id(persistent, name, version, make_current) -> optional<int>
  mounted_any = false
  FOR EACH archive KNOWN BUT NOT OPEN
    IF archive.header."level_name" matches name AND archive.header."level_ver" matches version
      mount it
      mounted_any = true
  IF mounted_any
    level_scan()                       # the mount may have added level folders

  result = index of the level whose folder equals name + separator   # case-insensitive
  IF make_current AND result EXISTS
    level_set(result)
  IF mounted_any
    on_assets_changed()                # the renderer must re-resolve what it cached
  RETURN result
```

**Notes** — levels ship one archive each and those archives are *not* mounted at startup:
the virtual filesystem knows of them and reads only their headers, and the body is mounted
the first time the level is asked for. That is why a level's archive header carries its
name and version as data — it is the index by which the mount is found. Unmounting never
happens; a session that visits five levels ends with five archives mounted.

Comparing the requested name against `folder` requires appending the path separator,
because a level's identity is its folder string *with* the trailing separator. That is the
one place the trailing separator convention is visible, and getting it wrong makes every
lookup fail.

## `GetArchiveHeader`

**Contract** — finds the configuration header of the archive carrying a given level name
and version, without mounting anything. Returns nothing if no archive claims that pair.
Used to read a level's metadata (its display name, its minimum engine version) before
deciding to load it.

## The load bracket

**Contract** — `LoadBegin` and `LoadEnd` are a reference-counted bracket around any span
that shows the loading screen. Only the transitions in and out of zero do anything.

```text
FUNCTION load_begin(persistent)
  load_depth = load_depth + 1
  IF load_depth == 1
    loaded = false
    phase_timer.restart()
    load_stage = 0

FUNCTION load_end(persistent)
  load_depth = load_depth - 1
  IF load_depth == 0
    report the phase's elapsed time and the process's memory use
    run the memory-statistics console command
    loaded = true
```

**Notes** — counted rather than boolean because loading nests: starting a level loads the
level, which starts the script layer, which may itself load. A boolean would be cleared by
the inner load and leave the outer one drawing nothing.

`loaded` is what the script-visible "is the application ready" predicate reports, and what
`LoadDraw` tests to stop drawing the loading screen. It is therefore the one flag that
distinguishes "a level exists" from "a level exists and is playable".

## `LoadStage`

**Contract** — advances the loading screen's progress by one step, optionally drawing a
frame. Re-reads the total step count each call, because it depends on the session type.

```text
FUNCTION load_stage(persistent, draw)
  REQUIRE a load is in progress
  IF the loading screen is not driving the frame loop itself
    report and restart the phase timer          # each stage is timed separately

  IF game type is single-player AND the alife flag is set
    max_load_stage = 18
  ELSE
    max_load_stage = 14

  loading_screen.show()
  loading_screen.update(load_stage, max_load_stage)
  IF draw
    load_draw()
  load_stage = load_stage + 1
```

**Notes** — **the step totals are hand-counted and are not derived from anything.** Eighteen
for a single-player level with the off-screen simulation, fourteen otherwise; the
difference is the four stages the alife simulation adds. Nothing verifies that the callers
actually call this the promised number of times, so the progress bar is approximately right
by maintenance rather than by construction. This is the honest answer to "why 18": somebody
counted. A rebuild should either derive the total from a declared stage list or drop the
proportion and show an indeterminate indicator.

The phase timer is reported per stage only when the loading screen is *not* being pumped by
the frame loop's load queue. When it is, every iteration would print a line.

## `LoadTitle`

**Contract** — sets the loading screen's stage caption from a localization key, and
optionally rolls a new gameplay tip. Then advances a stage.

```text
FUNCTION load_title(persistent, title_key, change_tip, map_name)
  IF title_key EXISTS
    loading_screen.stage_title = translate(title_key) + "..."
  ELSE IF NOT change_tip
    loading_screen.stage_title = ""

  IF change_tip
    picker = script function "loadscreen.get_tip_number"      (single-player)
             OR "loadscreen.get_mp_tip_number"                (otherwise)
    IF the script does not define it
      RETURN                                                  # no tip; see note
    n = picker(map_name)
    loading_screen.set_tip(translate("ls_header"),
                           translate("ls_tip_number") + n + ":",
                           translate("ls_tip_<n>" or "ls_mp_tip_<n>"))
  load_stage()
```

**Notes** — which tip is shown is **the script layer's decision, not the engine's**. The
engine asks a named Lua function for a number and then looks up three localization keys
built from it. That indirection exists so a modification can weight tips by level, or show
only tips relevant to what the player has not yet done, without an engine change. It also
means the engine must tolerate the function being absent, which it does by showing no tip
at all rather than by failing.

Returning early when there is no tip function **skips the stage advance**, so a build
without the script tip table counts one fewer loading stage than the progress total
assumes. A minor wart, visible as a progress bar that never quite fills.

## `LoadDraw` — a frame outside the frame loop

**Contract** — draws one loading-screen frame by opening a render bracket directly. Does
nothing once the load is finished.

```text
FUNCTION load_draw(persistent)
  IF loaded
    RETURN
  advance the device's frame number by one          # see note
  IF NOT device.render_begin()
    RETURN
  loading_screen.draw()
  device.render_end()
```

**Notes** — this is the escape hatch that lets a long, blocking load step still put pixels
on screen: any code that is about to do something slow calls `LoadTitle`, which lands here,
and one frame is drawn from inside the middle of the load. It is the *other* loading path,
the one for work that could not be broken into resumable steps for the frame loop's load
queue.

Incrementing the frame number by hand is what makes the once-per-frame guards elsewhere in
the engine — the camera matrix rebuild, the console's tip refresh — behave. Without it they
would consider themselves already done and the screen would not update. A rebuild that
drives loading from a coroutine on the real frame loop does not need any of this.

## `OnFrame`

**Contract** — joins the update phase at high priority. Rebuilds the two spatial
acceleration structures, advances the weather, and services the particle world's two
queues.

```text
FUNCTION on_frame(persistent)
  spatial_objects.update()
  spatial_physics.update()

  IF NOT paused OR a precache is running
    environment.on_frame()

  record the three particle counts for the statistics overlay

  WHILE particles_to_play is not empty
    take the last AND play it

  WHILE particles_to_destroy is not empty
    take the last
    IF it is locked
      log and BREAK                 # see note
    delete it
```

**Notes** — **the weather advances during a precache even though the game is paused.** The
precache spins the camera through the level with the simulation stopped; if the weather did
not advance with it, every shader whose appearance depends on the time of day would be
warmed at whatever moment the level happened to load, and the first real frame would
recompile them.

Both particle queues are drained **from the back**, so within one frame they run in reverse
order of request. Nothing depends on the order, but a rebuilder comparing behaviour should
know.

A **locked** particle system is one a caller is currently holding a reference into; deleting
it would leave that reference dangling. The loop `BREAK`s rather than skipping, so one
locked system defers every system behind it to the next frame. That is a deliberate
simplification — the queue is a stack and there is nowhere to put the skipped entry — and
its cost is bounded because a lock lasts at most a frame.

## `destroy_particles`

**Contract** — empties the particle world, in one of two strengths. `all` destroys every
active system; otherwise only those that declare themselves destroyable on a game load
survive the cut. Always clears both queues first. Asserts nothing is left pending.

```text
FUNCTION destroy_particles(persistent, all)
  clear particles_to_play
  drain particles_to_destroy completely        # none may be locked here; asserted
  IF all
    delete every active system
  ELSE
    snapshot the active set
    delete exactly those that answer yes to destroy_on_game_load
```

**Notes** — the two strengths are the difference between *shutting down* and *changing
level*. A system that answers no to `destroy_on_game_load` is one owned by something that
survives the level change — typically an interface effect or a weather effect — and killing
it would leave its owner holding nothing.

The active set is **snapshotted before deleting**, because each deletion removes its own
entry from that set and iterating it while it mutates is the bug this guards against.
Unlike the per-frame path, locked systems are asserted absent rather than tolerated: this
runs between levels, where nothing should be holding a reference.

## Application lifetime hooks

**`OnAppStart`** — loads the weather system's configuration and scans the level catalogue.
This is the first moment the engine knows what is installed.

**`OnAppEnd`** — unloads the weather, ends the current game type (which releases the
prefetched objects and every model), and releases the catalogue.

**`OnGameStart` / `OnGameEnd`** — called only when the *game type* changes, never on an
ordinary level change. Start raises the prefetch title and runs the prefetch; end clears the
object pool and drops every cached model.

**`OnAppActivate` / `OnAppDeactivate`** — empty here; the game module overrides them.

**`OnAssetsChanged`** — tells the renderer that the file namespace changed underneath it, so
that anything it resolved by path must be resolved again. Raised after an archive mount.

## Construction and destruction

**Contract** — on construction: attaches handlers for the four lifecycle events, joins the
update phase just above the highest priority band and both focus sequences, creates the
weather system, the shared shader-constant block, and the process's default sound scene.
Destruction reverses all of it in the opposite order.

**Notes** — the update-phase priority is deliberately one step below the top: the loading
screen's own frame handler registers at the very top, so that during a load the screen is
updated before the persistent layer touches anything.

## `IsMainMenuActive` / `MainMenuActiveOrLevelNotExist`

**Contract** — the two predicates the renderer and the console ask when deciding whether
there is a world behind the interface. The second is the "nothing worth drawing under this"
test: no level at all, or a menu covering it.

## `DumpStatistics`

**Contract** — prints the three particle counts — starting, active, being destroyed — to the
statistics overlay, then resets them for the next frame. The source marks this as belonging
in the particle module rather than here, and it does.
