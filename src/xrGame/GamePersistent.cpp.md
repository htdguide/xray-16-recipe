# src/xrGame/GamePersistent.cpp

> The game module's process-lifetime object: it brings the game layer up and down around the engine's own startup, owns the main menu and loading screen, drives the intro chain, plays the weather system's ambient sounds and wind gusts, and animates the depth-of-field effector.

**Needs** — [`GamePersistent.h`](GamePersistent.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`Level.h`](Level.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`Actor.h`](Actor.h.md) · [`Spectator.h`](Spectator.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`game_sv_single.h`](game_sv_single.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`ui/UIGameTutorial.h`](ui/UIGameTutorial.h.md) · [`ui/UILoadingScreen.h`](ui/UILoadingScreen.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: lifecycle ordering, timer-driven scheduling and vector interpolation; nothing here touches a byte layout

## Purpose

The engine owns a frame loop but knows nothing about a *game*. This object is the game
module's answer to that: one instance that exists for the whole process, from before a
level is loaded until after the last one is unloaded. It has three distinct jobs that
share nothing but their lifetime, and a rebuild is free to split them:

1. **Bring-up and tear-down ordering** for everything the game layer owns globally — the
   material interaction table, the UI core, the main menu, the loading screen, the
   game-wide script globals — with the constraint that the engine's own startup runs
   *between* two halves of it.
2. **The atmosphere driver.** The weather system supplies a *description* of the current
   ambience (a set of sound channels with random intervals, a set of one-shot effects
   with wind parameters); this file turns that description into actual playing sounds,
   particle effects and a wind-strength envelope, once per frame.
3. **The screen-state machine** around a session: the intro chain, the pause behaviour
   on window focus change, quick-load, and the depth-of-field effector.

It is also where the game module answers the engine's question "what is a level?" — the
only place `CLevel` is constructed.

## State

```text
RECORD AmbientState                # drives the atmosphere from the weather description
  sound_next_time : map<int,int>   # per ambient sound channel: absolute time the next one may start
  effect_next_time : int           # earliest time a new one-shot ambient effect may start
  effect_stop_time : int           # when the running effect must be cut
  particles        : optional<ParticleEffect>   # at most one ambient effect plays at a time

RECORD WindBlastEnvelope           # the gust an ambient effect superimposes on the weather's wind
  start     : real   # time the gust begins
  in_time   : real   # time the ramp-up ends
  end       : real   # time the ramp-down begins
  out_time  : real   # time the gust is fully gone
  active    : bool
  # invariant: start <= in_time <= end <= out_time; strength is interpolated
  # start..in_time, held in between, and interpolated back to zero end..out_time

RECORD DofState                    # depth of field, four points of one animation
  dest     : vec3   # where it is heading
  current  : vec3   # what the renderer is given
  from     : vec3   # where the current move started
  original : vec3   # the console-configured resting value
  pickable : bool   # true while the crosshair's aim distance drives it instead

RECORD SessionState
  intro_step   : optional<callback>  # the next stage of the intro chain, or none when finished
  intro        : optional<Sequencer> # at most one intro sequence exists at a time
  demo_file    : optional<Reader>    # the playlist when started in demo mode
  time_to_change : int               # when the demo playlist advances
```

Invariants worth stating: at most one ambient effect and at most one intro sequence exist
at any moment, and both are torn down by the same object that created them. The four
depth-of-field points are always mutually consistent — setting the base sets all four, so
there is never a partially-initialized animation.

## `CreateLevel` / `DestroyLevel`

**Contract** — construct and destroy the game's level object. This is the entire reason
the engine can be built without knowing what a level contains: the engine asks for one by
interface and the game module supplies its concrete kind. A rebuild should keep the seam;
it is what lets the dedicated server and the editor swap the level implementation.

## `OnAppStart`

**Contract** — the game layer's half of process startup, run around the engine's. Loads
the material interaction table, starts the game-wide script globals, creates the UI core,
the main menu and a loading screen, then calls the engine's own startup, then the
platform-specific extras.

**Invariants** — the material table must be fully loaded before anything that can
reference a material exists, and it must be loaded *alone*: it is the one initializer in
this sequence that is not safe to run concurrently with the others. The loading screen
kind is chosen by whether this process renders at all — a dedicated server installs a
loading screen that draws nothing rather than branching at every call site.

```text
FUNCTION on_app_start()
  load_material_table()                   # must complete before any user of it starts
  globals_task = SPAWN init_game_globals()  # script-side globals; independent of the above
  create_ui_core()
  create_main_menu()
  loading_screen = IF this process renders THEN real one ELSE a no-op one
  inherited_on_app_start()                # the engine brings up its own world
  AWAIT globals_task
```

**Notes** — the globals initializer is run concurrently on one platform and serially on
the others. That is a workaround for a platform-specific crash, not a decision: a rebuild
should either make the initializer genuinely independent and always run it concurrently,
or always run it serially.

## `OnAppEnd`

**Contract** — the exact reverse of startup: close the menu if it is open, destroy the
loading screen and menu, let the engine tear down, then drop the script globals, the UI
core and the material table. The ordering is the point — script globals may hold handles
to UI objects, so they die first; the material table is referenced by everything, so it
dies last.

## `Disconnect`

**Contract** — leave a session without leaving the process. Destroys the ambient particle
effect, lets the engine tear the level down, silences every playing sound emitter, and
resets the game type to "no game". Silencing is explicit because sounds outlive the
objects that started them: a looping ambient bed started by a level that no longer exists
would otherwise keep playing over the menu.

## `OnGameStart` / `OnGameEnd`

**Contract** — bracket one session. Start re-derives the game type from the session
parameters; end releases the two shared per-session caches the stalker animation layer
owns (a pose-data store and a velocity table). Those caches are global rather than
per-creature because animation data is shared across all creatures of a kind; they are
dropped between sessions so a new level's animation set does not inherit the old one's.

## `UpdateGameType`

**Contract** — parse the session's game-type string into the enumeration, and switch the
active **key binding group** accordingly. Single player and multiplayer have separate
binding sets, because the same physical key means different things in each; the group is
a global the input layer reads, so it must be set whenever the game type changes and not
only at startup.

**Notes** — two game-type codes exist twice, once with the value the first game shipped
and once with the value the later ones use. The translation from code to name collapses
the pair. This is pure compatibility with two generations of shipped data and a rebuild
must reproduce it if it is to read either game's server announcements.

## `GameTypeToString` / `GameTypeToStringEx`

**Contract** — map a game type to a name, in a long form and a short form. Both forms are
frozen: the short names appear in server browser entries and console commands, the long
ones in configuration. The "Ex" variant additionally folds the two legacy codes onto
their modern equivalents before mapping.

## `OnFrame`

**Contract** — the game layer's per-frame work, run inside the engine's frame. Advances
the intro chain, retires finished tutorial sequences, ticks the scheduler, updates the
ambience, advances the demo playlist and animates the depth of field. Returns early — and
in particular does *not* tick the scheduler — when no level is loaded or the level is not
yet ready.

**Invariants** — the scheduler and the ambience are ticked only while the game is not
paused; the intro chain and the menu run regardless, which is what makes the menu usable
over a paused session.

```text
FUNCTION on_frame()
  IF precache countdown reached its titling frame AND no intro step is pending THEN
    show_load_title()
    intro_step = game_loaded          # arm the post-load stage of the chain
  retire_finished_tutorials()
  every 200th frame: release cached UI shaders   # bounded-size cache, swept not counted
  IF this process renders THEN
    IF an intro step is pending THEN run it
    ELSE IF no intro is playing AND precaching has finished THEN stop the loading screen
  IF the menu is not active THEN release its internal allocations
  IF no level OR level not ready THEN RETURN

  IF paused THEN
    update_camera_of_the_view_entity()     # so a paused game can still be looked around
  inherited_on_frame()
  IF NOT paused THEN
    scheduler.update()
    weathers_update()
  advance_demo_playlist_if_due()
  update_dof()
```

**Notes** — the paused branch is the interesting one. While paused the engine does not
tick anything, yet the camera must still follow the view entity, because a demo playback
can be paused and looked around, and because the free-camera debug mode moves the actor
with the world stopped. The debug branch there forces one update of the actor and every
object it carries with a zero time delta — an explicit "advance these, and only these,
by no time at all", which exists so that a no-clip camera does not desynchronize the
entity's attachments from its transform.

## `WeathersUpdate`

**Contract** — turn the current weather's *ambient description* into playing sound and
one visible effect. Runs once per unpaused frame while a level is loaded and this process
renders. Allocates a particle effect at most once per effect cycle.

**Invariants** — each ambient sound channel has its own next-start time, and a channel's
next start is always at least the sound's own length past its start, so a channel never
overlaps itself. Ambient effects play only outdoors; indoor-ness is decided from the
hemisphere lighting the view entity receives, not from the sector topology, because the
level does not mark interiors.

```text
FUNCTION weathers_update()
  indoor = view entity's received hemispheric light < a small threshold
  ambient = current environment's ambient description
  IF ambient exists THEN
    FOR EACH channel, index IN ambient.sound_channels
      IF sound_next_time[index] is unset THEN
        sound_next_time[index] = now + channel.first_delay()      # stagger the first one
      ELSE IF now > sound_next_time[index] THEN
        sound = channel.pick_random()
        play sound at a random bearing around the camera, at the channel's distance,
          raised above the listener                                # a sky-ish placement
        sound_next_time[index] = now + sound.length + channel.gap()

    IF NOT indoor AND no effect is playing AND now > effect_next_time THEN
      effect = ambient.pick_random_effect()
      IF effect exists THEN
        effect_next_time = now + ambient.effect_gap()
        effect_stop_time = now + effect.life_time
        arm the wind envelope from effect's blast in/out times and life time
        particles = create and play effect.particles at the camera plus its offset
        play effect.sound at the same place
        wind gust target = effect.blast_strength, direction = effect.blast_direction

  advance_wind_envelope()
  IF indoor OR now >= effect_stop_time THEN stop the particles and clear the gust factor
  IF the particles have finished playing THEN destroy them
```

**Notes** — the wind envelope is a spherical interpolation of the *direction* and a linear
interpolation of the *strength*, ramping in over the effect's declared in-time, holding,
and ramping out after the effect's life ends. Direction is interpolated as a rotation
rather than as a vector so a gust that reverses does not pass through zero wind. The
ramp-out begins at the effect's end whether or not the particles have stopped, so the
audible and visible gust outlive the wind slightly — which is the intended feel.

A detail a rebuild must not lose: when the wind strength is currently zero, the gust's
*start* direction is taken to be the gust's own direction rather than the environment's
current direction. With no wind there is no meaningful current direction to rotate away
from, and interpolating from a stale one produces a visible swing at the start of every
gust after a calm.

## `OnEvent`

**Contract** — handles two deferred engine events. **Quick load**: unpause, dismiss every
open dialog, reset the main in-game window and the PDA, stop any running tutorial, drop
every object in the level, and restart the single-player simulator from the named save.
**Demo start**: issue the console command that begins demo playback and arm the playlist
timer. Anything else falls through to the engine.

**Invariants** — quick load removes the level's objects *before* the simulator restarts,
not after. The restart re-spawns from the save's records; leaving the old objects alive
would give two live instances of the same entity identifier, which the registries forbid.

## The intro chain

**Contract** — a four-stage chain, each stage a one-shot that arms the next: logo intro →
(after precache) game loaded → game intro → done. Each stage owns at most one UI sequence
and deletes it in its own completion callback. Stages are skipped by command-line switch
or by circumstance — no logo when a level is being loaded directly, no "game loaded"
prompt unless the loading screen actually asked for a keypress, no game intro unless the
session is a *new* game rather than a load.

**Notes** — the chain is expressed as a rearmable "next step" slot rather than a state
enumeration. A rebuild should read it as a small explicit state machine; the important
content is the *conditions*, which are player-visible: the logo plays only when the
process comes up to the menu, and the story intro plays only on a new game.

## `OnAppActivate` / `OnAppDeactivate`

**Contract** — pause on losing window focus and resume on regaining it, unless the
"always active" setting is on. Multiplayer never pauses the simulation on focus loss —
it cannot, the server keeps running — but it still mutes.

**Invariants** — the pause state that existed *before* deactivation is remembered, so
regaining focus does not un-pause a game the player had paused deliberately. The entry
flag guards against a deactivation arriving without a matching activation, which the
windowing layer can deliver at startup.

## `CanBePaused`

**Contract** — true for single player and for demo playback; false otherwise. The single
authority on whether the pause key does anything.

## Depth-of-field effector — `SetBaseDof`, `SetEffectorDOF`, `RestoreEffectorDOF`, `SetPickableEffectorDOF`, `GetCurrentDof`, `UpdateDof`

**Contract** — a small animated three-component value (near, focus, far) handed to the
renderer each frame. Callers push a target; the value moves toward it over a fixed time
and stops. A separate mode makes the focus track the crosshair's aim distance instead,
and while that mode is on, pushed targets are ignored.

**Invariants** — the animation always converges: each component is clamped to the
interval between where the move started and where it is going, in whichever order those
two happen to be. Without the clamp a large time step would overshoot and oscillate.

```text
FUNCTION update_dof()
  IF pickable THEN
    focus = the HUD ray query's current range        # what the crosshair is pointing at
    dest = (focus + near_offset, focus, focus + far_offset)
    from = current
  IF current is already at dest THEN RETURN
  step = (dest - from) * (frame_delta / transition_seconds)
  current = current + step
  clamp each component of current between from and dest
```

**Notes** — the transition takes a fifth of a second and the near/far offsets around the
picked distance come from a configuration section with defaults of a large negative and a
large positive. Those numbers are tuning, not structure; what is load-bearing is that the
transition is time-based rather than frame-based, so the effect looks the same at any
frame rate.

## `DumpStatistics`

**Contract** — append the game layer's per-frame numbers to the engine's debug overlay,
including the physics world's own, and raise a performance alert when a budget is
exceeded. Debug-only; a rebuild may omit it.

## `OnSectorChanged`

**Contract** — tell the main in-game window the view entity moved to a different
visibility sector. The UI uses it to re-derive which level region the player is in for
the map and the location caption.

## `OnAssetsChanged`

**Contract** — after the virtual filesystem notices changed game data, re-scan the
localization string table so text edits take effect without a restart. A development
convenience whose only requirement is that the string table can be rebuilt in place.

## `OnRenderPPUI_query` / `OnRenderPPUI_main` / `OnRenderPPUI_PP`

**Contract** — pure delegation to the main menu: whether the post-process UI pass should
run at all, and the two halves of it. The menu is drawn through the post-process chain so
that it can be blurred over a live scene.

## Notes

The demo playlist is a text file of comma-separated tuples — server parameters, client
parameters, demo name, dwell seconds — replayed in a cycle and driven by deferred engine
events rather than direct calls, because each entry implies a disconnect and a level
change. It exists for unattended benchmarking and attract-mode loops. A rebuild can drop
it without affecting anything else.

The per-200-frames sweep of cached UI shaders is unexplained in the source: the cache has
no size bound, so the sweep is what keeps it from growing across a long session. The
number 200 has no discoverable justification.
