# src/xrEngine/device.cpp

> The frame loop: what happens between one presented image and the next, and the clocks everything else reads.

**Needs** — [`device.h`](device.h.md) · [`Render.h`](Render.h.md) · [`pure.h`](pure.h.md) · [`Stats.h`](Stats.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_input.h`](xr_input.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`editor_base.h`](editor_base.h.md) · [`defines.h`](defines.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`device.h`](device.h.md)
**Tier floor** — T1: a hard frame budget, device-lost handling, and matrices published by reference into device-facing code

## Purpose

This is the spine. It owns the three clocks the whole engine reads, the camera and its
matrices, the ordered lists of per-frame callbacks, and the loop that runs them. Everything
else in the engine is something this file calls or something that reads what this file
publishes.

The parts of the device that are about *bringing it up* — window creation, resolution
selection, device reset, overlay initialization — live in the sibling `Device_*.cpp` files.
What is here is the steady state.

## State

**Clocks.** Three, and the distinction between them is load-bearing.

```text
timer         : PausableTimer   # per-frame delta source; pauses with the game
timer_global  : PausableTimer   # global game time; pauses with the game; scriptable
timer_mm      : Timer           # never pauses, never scales: real wall time
timer_mm_delta: int             # offset aligning timer_mm with the platform's tick count
```

`timer` and `timer_global` both stop when the game pauses and both are scaled by the time
factor, so game logic slows and stops with them. `timer_mm` does neither: it is what the
inactive-time accounting and the asynchronous time queries are built on, because a menu or
an alt-tab must not appear to the network layer as time standing still.

**Published time.** Read by nearly every file in the engine.

```text
frame_number      : int         # increments once per frame; a universal "have I done this
                                #   already this frame" stamp
time_delta        : real        # seconds; *smoothed*. what game logic integrates over
time_delta_real   : real        # seconds; this frame's actual measured elapsed time
time_global       : real        # seconds since start, paused and scaled
time_delta_ms     : int         # milliseconds; the integer form of the delta
time_global_ms    : int         # milliseconds; the integer form of global time
time_continual_ms : int         # real ms since start, minus every ms spent inactive
```

**Invariants** — `time_delta` and `time_delta_ms` are derived differently and do *not* agree:
the float form is the exponentially smoothed delta, the integer form is the exact difference
between two consecutive global-time readings. Code that mixes them accumulates drift. This
is a wart of the original, and a rebuild should derive both from one source.

**Camera and matrices.** Published, read directly by the renderer and by anything that culls.

```text
camera_position, camera_direction, camera_top, camera_right : vector3
view, inverse_view, projection, full_transform, inverse_full_transform : matrix4
field_of_view, aspect : real

# and a parallel saved set, frozen at the end of each frame's render phase
camera_*_saved, view_saved, projection_saved, full_transform_saved
```

The saved copies exist because the render phase and the next frame's update phase overlap
in a threaded build: the update phase begins moving the camera while worker tasks from the
frame just drawn still need the matrices that frame was drawn with. Capturing them at
render end gives those readers a stable set.

**Window and device.**

```text
window                  : platform window
window_bounds           : rectangle    # the real window, including decoration
window_client           : rectangle    # the drawable area
width, height           : int          # render target size
half_width, half_height : real         # published because screen-space maths needs them
is_ready                : bool         # a graphics device exists
is_active               : bool         # running at full rate
is_in_focus             : bool         # the platform says we have focus
nearer                  : bool         # the projection is biased toward the viewer
```

`is_active` and `is_in_focus` are separate because the "always active" flag lets a window
that has lost focus keep running at full rate.

**Callback registries.** Priority-ordered lists; a subsystem joins one to be called.

```text
seq_render         # the render phase, in priority order
seq_frame          # the update phase, on the main thread
seq_frame_mt       # the update phase, on a worker, concurrent with rendering
seq_parallel       # one-shot closures, run once on a worker then cleared
seq_app_activate / seq_app_deactivate    # focus transitions
seq_app_end        # shutdown
seq_device_reset   # a graphics device was rebuilt: re-create everything device-bound
seq_ui_reset       # the resolution changed: re-lay-out the interface
```

**Statistics and precache.**

```text
render_total, engine_total : Timer   # the render phase and the update phase
fps, render_fps, tps       : real    # all three exponentially smoothed
precache_frame, precache_total : int # frames remaining and the original count
```

## The frame limiter

Two console-set caps, in frames per second:

```text
fps_limit          = 501      # in game: effectively uncapped
fps_limit_in_menu  = 60       # paused or no level: cap hard
dedicated_rate     = 100      # a dedicated server's tick rate
```

**Notes** — 501 rather than 500 so that a cap expressed as `1000 / limit` rounds to one
millisecond and the limiter effectively never sleeps; the value is "off" wearing a number.
The menu cap exists because a menu renders in microseconds and would otherwise spin a core
at thousands of frames per second.

## `ProcessFrame`

**Contract** — one iteration of the loop. Everything else in this file is a step of it.

```text
FUNCTION process_frame(device)
  IF NOT before_frame(device)
    RETURN                                 # not ready, or a load step ran instead

  start = timer_global.elapsed_ms

  frame_move(device)                       # the update phase
  on_camera_updated(device)                # rebuild the matrices the render phase reads

  parallel = SPAWN
      run every queued one-shot closure, then clear the queue
      run the worker-side update sequence
  END SPAWN

  do_render(device)                        # the render phase, concurrent with the above

  AWAIT parallel

  elapsed = timer_global.elapsed_ms - start
  budget  = 1000 / (dedicated server ? dedicated_rate
                    : (paused OR no level) ? fps_limit_in_menu
                    : fps_limit)
  IF elapsed < budget
    sleep(budget - elapsed)
  IF NOT is_active
    sleep(1)                               # yield hard when unfocused
```

**Notes** — the shape is the decision. The update phase finishes *before* rendering starts,
and only then is a second block of work — the deferred closures and the worker-side update
sequence — run concurrently with rendering. So the engine is not pipelined: a frame's
simulation and its drawing do not overlap. What overlaps is drawing and the *leftovers* of
simulation that were explicitly marked safe to run beside it. A rebuild attempting real
pipelining has to answer what the saved matrix set answers here, but for the whole object
graph rather than just the camera.

`on_camera_updated` sits between the two phases because the update phase is what moves the
camera and the render phase is what consumes the matrices. Running it early would publish
last frame's view.

The limiter sleeps on wall time measured with the *global* timer, which pauses. A paused
game therefore measures zero elapsed and sleeps the whole budget, which is the desired
behaviour and is arrived at accidentally.

## `BeforeFrame`

**Contract** — the gate. Answers whether a normal frame should run at all, and if not, does
whatever should happen instead.

```text
FUNCTION before_frame(device) -> bool
  IF no graphics device yet
    sleep(100)                             # nothing useful to do; do not spin
    RETURN false

  statistics gathering = whether the statistics flag is set

  IF the load queue is not empty
    run the front load step once
    IF it reports itself finished
      remove it
    persistent.load_draw()                 # keep the loading screen on screen
    RETURN false                           # no ordinary frame this time

  RETURN true
```

**Notes** — this is how a level load is driven. Loading is not a blocking call: it is a
queue of steps, one of which runs per iteration of the main loop, so the window keeps
pumping events and the loading screen keeps drawing. Each step reports whether it is done,
which lets one step span many iterations. A rebuild with coroutines or async has a better
tool for this; the requirement is only that the platform's event queue keeps being served
while a level loads.

## `FrameMove` — the update phase

**Contract** — advances the clocks and runs the main-thread update sequence.

```text
FUNCTION frame_move(device)
  frame_number = frame_number + 1
  time_continual_ms = timer_mm.elapsed_ms - total_inactive_ms

  time_delta_real = timer.elapsed_sec
  IF time_delta_real is not a finite number
    time_delta_real = a tiny positive epsilon      # see note
  timer.restart()

  IF the constant-frame-rate flag is set
    time_delta = 0.033 ; time_global += 0.033
    time_delta_ms = 33 ; time_global_ms += 33
  ELSE IF paused
    time_delta = 0
  ELSE
    time_delta = 0.1 * time_delta + 0.9 * time_delta_real   # smooth
    clamp time_delta INTO [epsilon, 0.1]                    # never report worse than 10 fps
    time_global = timer_global.elapsed_sec
    previous = time_global_ms
    time_global_ms = timer_global.elapsed_ms
    time_delta_ms = time_global_ms - previous

  publish time_delta_real to the overlay toolkit and begin its frame

  engine_total.begin()
  seq_frame.process()                       # every main-thread per-frame subscriber
  engine_total.end()

  end the overlay toolkit's frame
```

**Notes** — four decisions.

**The delta is smoothed, not fixed.** Ninety percent of the new measurement, ten percent of
the old, which absorbs the jitter of operating-system scheduling at a cost the source
estimates at about seven percent error in the worst case. This is *not* the fixed-timestep
accumulator the system requirements describe — that lives in the simulation layer, which
consumes this delta and steps its own fixed increments against it. The engine's frame
pacing is variable; the world's advance is not.

**The delta is clamped to a tenth of a second.** A frame that really took longer — a level
streaming hitch, a breakpoint, a laptop resuming from sleep — is reported as a tenth of a
second, so the simulation advances slowly rather than teleporting everything. The
consequence is that the game runs in slow motion during a stall rather than skipping. That
is the right trade for a shooter with physics and is the single most important line in the
function.

**A non-finite delta is replaced rather than propagated.** The clock can return a nonsense
value across a suspend or a frequency change, and one such value would poison every
integrator in the engine within a frame.

**The constant-frame-rate flag feeds a literal 33 ms** and advances global time by the same
amount, decoupling the whole engine from real time. It is how a deterministic replay or a
repeatable benchmark is run. Both the float and integer forms are set consistently here,
which is the one place they agree.

## `DoRender` — the render phase

**Contract** — draws one frame if the device is usable, and otherwise only services the
overlay's extra windows. Measures itself with a *separate* timer from the published one, so
the published figure is not disturbed by the measurement.

```text
FUNCTION do_render(device)
  IF dedicated server
    RETURN
  measure FROM here
  IF is_active AND render_begin(device)
    seq_render.process()                   # the world, then every overlay
    calc_frame_stats(device)
    statistics_overlay.show()
    the overlay toolkit renders; its draw data goes to the backend
    update the overlay toolkit's extra platform windows
    render_end(device)                     # presentation happens inside
  ELSE
    update the overlay toolkit's extra platform windows
  render_total = the measured interval
```

**Notes** — the overlay's extra windows are updated on *both* paths. A debug tool torn off
into its own window must keep repainting even when the main window is not drawing, or it
appears frozen when the game is minimized.

## `RenderBegin`

**Contract** — the device-loss protocol, stated as three cases.

```text
FUNCTION render_begin(device) -> bool
  IF dedicated server
    RETURN true                            # there is nothing to draw, but the frame proceeds
  CASE renderer.device_state()
    NORMAL      -> fall through
    LOST        -> sleep(33) ; RETURN false      # cannot draw; wait a frame and re-ask
    NEED_RESET  -> reset(device) ; RETURN false  # rebuild, draw nothing this frame
  renderer.begin()
  rendering_in_progress = true
  RETURN true
```

**Notes** — the 33 ms sleep in the lost state is a frame at thirty per second: the loop must
not spin while the device is gone (a fullscreen application on another display, a driver
resetting), but it must keep pumping events so the platform can tell it the device is back.

`rendering_in_progress` is the process-wide flag every renderable's destructor asserts
against (see [`IRenderable.cpp`](IRenderable.cpp.md)). Its whole purpose is to catch an
object being destroyed from inside the render phase.

## `RenderEnd`

**Contract** — ends the frame, presents, runs the precache countdown, and freezes the saved
camera set.

```text
FUNCTION render_end(device)
  IF dedicated server
    RETURN

  IF precache_frame > 0
    sound.master_volume = 0                # silence while the world is being warmed up
    precache_frame = precache_frame - 1
    IF precache_frame reaches 0
      renderer.update_gamma()
      release the precache light
      sound.master_volume = 1
      renderer.drop every texture not touched during precache
      compact the allocator and report memory in use
      IF the game is the one that wants it AND "always active" is off
        IF the window does not have focus
          pause(timers and sound, reason = "application start")

  rendering_in_progress = false
  renderer.end()                           # presentation

  freeze camera_*_saved, view_saved, projection_saved, full_transform_saved
```

**Notes** — the precache is the reason a level does not stutter for its first ten seconds. It
spins the camera through a full circle over its frames (see `OnCameraUpdated`) while the
renderer is in deferred-load mode, so every shader, texture and mesh the level can show is
requested and uploaded before the player is given control. It is measured in *frames*, not
time, because what must complete is a number of full render passes, not a duration.

Muting during the precache is not cosmetic: the spinning camera moves the listener through
the whole level and would play every ambient sound in it at once.

Dropping untouched textures at the end is the payoff — the set of textures actually reached
during a full spin is the level's real working set, and everything else can go.

The pause-on-focus-loss at the end of the precache is marked as a hack in the source and is
game-specific. The honest reading: a level load takes long enough that the player has often
alt-tabbed away, and starting the simulation into an unfocused window loses them the first
seconds of the level.

## `PreCache`

**Contract** — arms the precache for a number of frames. Disarmed entirely on a dedicated
server and when the renderer is running a reference device, where warming caches is
meaningless. Creates a small white light at the camera so the warm-up passes actually
exercise the lit shader paths rather than only the unlit ones, and raises the loading screen
if it is not already up.

```text
FUNCTION precache(device, frames, wait_for_user_input)
  IF dedicated server OR the renderer is a reference device
    frames = 0
  precache_frame = precache_total = frames
  IF frames > 0 AND no precache light AND a level is loaded AND the load queue is empty
    create a shadowless white light at the camera, range 5, and enable it
  IF frames > 0 AND the loading screen is not up
    raise it, optionally waiting for the player to acknowledge
```

**Notes** — the light is created only when the *load queue is empty*, that is, when loading
has finished and this is the warm-up rather than a mid-load precache. Its range of five
metres is enough to make the lighting path run without lighting anything the player will
see.

## `OnCameraUpdated`

**Contract** — rebuilds the derived matrices from the camera, at most once per frame. During
a precache it *overrides* the camera direction, sweeping it through a full turn across the
precache's frames.

```text
FUNCTION on_camera_updated(device)
  IF already done this frame
    RETURN
  IF precache_frame > 0
    angle = 2pi * precache_frame / precache_total
    camera_direction = (sin angle, 0, cos angle), normalized
    camera_top   = up
    camera_right = cross(camera_top, camera_direction)
    view = camera looking along camera_direction from camera_position
  inverse_view        = invert(view)
  full_transform      = projection * view
  inverse_full        = invert(full_transform)
  renderer.on_camera_updated()
  renderer.set_cache_xform(view, projection)
  mark this frame done
```

**Notes** — the once-per-frame guard is a frame-number stamp, the same pattern the whole
engine uses instead of dirty flags. It is needed because both the frame loop and several
callers that want fresh matrices mid-frame call this.

The precache sweep runs the camera *backwards* through the circle (the factor counts down
with the remaining frames), which is irrelevant to what it warms.

## `CalcFrameStats`

**Contract** — updates the three smoothed rates once per frame. All three use the same
coefficient.

```text
FUNCTION calc_frame_stats(device)
  render_total.frame_end()
  IF time_delta_real > epsilon
    fps = 0.7 * fps + 0.3 * (1 / time_delta_real)
    IF render_total.result > epsilon
      tps        = 0.7 * tps        + 0.3 * (triangles_this_frame / (render_total.result * 1000))
      render_fps = 0.7 * render_fps + 0.3 * (1000 / render_total.result)
  render_total.frame_start()
```

**Notes** — thirty percent of the new sample settles in roughly ten frames, which is fast
enough to see a stall and slow enough to read. The displayed frame rate is therefore never
the instantaneous one, and a rebuild that shows the raw value will be told its numbers are
noisier than the original's.

The three rates measure different things and are often confused: `fps` is the whole loop,
`render_fps` is what the frame rate would be if only the render phase existed, and `tps` is
millions of triangles per second. The gap between the first two is the update phase's cost.

## `ProcessEvent`

**Contract** — handles the platform events the device itself cares about; everything else is
forwarded to the editor overlay, which owns input translation.

```text
FUNCTION process_event(device, event)
  CASE event IS a display change (orientation, connected, disconnected)
    re-enumerate the available video modes
    IF the affected display is ours AND it was not merely a connection
      reset the device
    ELSE
      re-apply the window properties

  CASE event IS a window event
    find the overlay viewport belonging to that window; ignore the event if there is none
    moved         -> if it is our window, recompute the window rectangles;
                     tell the viewport to move
    display changed -> record which display we are now on
    resized       -> if it is our window, recompute the rectangles
    size changed  -> recompute the rectangles; if the size really differs from the
                     configured one, adopt it and reset the device; tell the viewport
    close         -> tell the viewport; if it is our window, queue a disconnect and a quit
  forward the event to the editor overlay
```

**Notes** — the size-changed handler compares against the *configured* mode before resetting.
Without that test, the reset it performs would itself generate a size-changed event and the
device would reset forever.

Closing the main window queues *two* deferred events, disconnect then quit, rather than
shutting down directly: a multiplayer session must leave the server cleanly, and neither can
run from inside an event handler.

A display being disconnected resets the device; a display being connected does not — gaining
a monitor changes nothing about the one being drawn to.

## `Run`

**Contract** — final startup. Calibrates the offset between the engine's own monotonic clock
and the platform's tick counter, then makes the window visible.

```text
FUNCTION run(device)
  time_global_ms = 0
  spin until the platform's tick counter changes    # align to a tick boundary
  timer_mm_delta = platform tick count - our own elapsed ms

  hide the window, re-apply its properties, show it, raise it
```

**Notes** — the spin-until-tick is a calibration: reading both clocks at an arbitrary moment
gives an offset uncertain by up to one tick of the coarser one, and waiting for the coarse
clock to advance removes that uncertainty. The offset makes the asynchronous time query —
the one the network layer and sound streaming use, which must be callable from any thread
without touching the pausable timers — agree with the game's own global time.

The hide-then-show is called a workaround for the windowing library in the source. A rebuild
should try without it.

## `Pause`

**Contract** — pauses or resumes, independently for timers and for sound, with a reason
string used only for diagnostics and for one exception.

```text
FUNCTION pause(device, on, pause_timers, pause_sound, reason)
  IF benchmarking OR dedicated server
    RETURN                                  # neither may ever pause

  IF on
    IF not already paused
      show the "paused" banner unless the editor is open, or the reason is the
        debug no-clip pause
    IF pause_timers AND the game permits pausing right now
      pause the pause manager
    IF pause_sound
      remember how many sound emitters were suspended
  ELSE
    IF pause_timers AND currently paused
      time_delta = a tiny positive value      # see note
      resume the pause manager
    IF pause_sound AND emitters were suspended
      resume them
```

**Notes** — resuming forces the frame delta to a tiny positive value rather than leaving it
at the zero a paused frame produced. The next frame's smoothing reads that value, and a zero
would not be smoothed away for several frames, during which every rate in the game would
read as stalled.

The pause manager is consulted, not commanded: the game module can refuse a pause — during a
cutscene, during a networked match — and the timer half is then skipped while the sound half
still happens.

The banner suppression is the only use of the reason string that changes behaviour: the
editor and the debug fly-through pause the world without telling the player it is paused.

Benchmarking and dedicated servers cannot pause at all, because in both cases the wall-clock
progression *is* the measurement.

## `OnWindowActivate`

**Contract** — the focus transition. Grabs or releases the pointer, decides whether the
engine keeps running at full rate, runs the activation sequences, pauses the worker pool,
and accounts the time spent inactive.

```text
FUNCTION on_window_activate(device, window, activated)
  IF the editor is fully open
    IF the window is not ours
      forward the transition to the editor and RETURN

  input.grab(activated AND not a dedicated server)
  is_active = activated OR the "always active" flag is set

  IF activated differs from is_in_focus
    is_in_focus = activated
    IF gained focus
      resume the worker pool
      seq_app_activate.process()
      total_inactive_ms += timer_mm.elapsed_ms - inactive_since
    ELSE
      inactive_since = timer_mm.elapsed_ms
      seq_app_deactivate.process()
      pause the worker pool
```

**Notes** — the order reverses between the two directions, and deliberately: on gaining focus
the workers resume *before* the subscribers run, so a subscriber may queue work; on losing
focus the subscribers run *before* the workers are paused, so anything they queue is drained
rather than stranded.

Inactive time is accumulated against the never-pausing real clock, and subtracted to give
"continual" time — real time the application was actually in the foreground. That is the
clock a network session must be timed against: a player who alt-tabs for a minute has not
been lagging for a minute.

`is_active` is separate from `is_in_focus` so that the "always active" flag keeps the frame
loop at full rate while the focus bookkeeping still runs correctly.

## `time_factor`

**Contract** — scales both pausable clocks, and the sound system's playback rate with them,
so slow motion sounds slow. A command-line switch opts the sound out, for recording.

**Invariants** — both clocks always carry the same factor; reading it asserts they agree.

## `AddSeqFrame` / `RemoveSeqFrame`

**Contract** — join or leave the update phase. A subscriber declares whether it is safe to
run on a worker concurrently with rendering. Worker-side subscribers join at high priority,
main-thread ones at low — so the concurrent work starts as early as possible in its own
sequence, and the main-thread work runs after whatever else already registered. Removal
tries both lists, since the caller need not remember which it joined.

## `CLoadScreenRenderer`

**Contract** — while active, joins both the update and render sequences at priority zero and
pumps the loading screen. Starting it tells the persistent layer to raise the screen and
begin a load; stopping it lowers the screen and ends the load. Idempotent on stop.

```text
FUNCTION start(renderer, wait_for_user_input)
  join seq_frame and seq_render at priority 0
  registered = true ; needs_user_input = wait_for_user_input
  persistent.show_loading_screen(true)
  persistent.load_begin()

FUNCTION on_frame(renderer)   -> persistent.load_stage(no wait)
FUNCTION on_render(renderer)  -> persistent.draw_loading_screen()
```

**Notes** — priority zero puts it ahead of everything in both sequences, which is what makes
the loading screen the first thing drawn and the first thing updated while a level comes up.
It unregisters itself when stopped rather than testing a flag every frame, the same pattern
the console uses.

## Script surface

**Contract** — the device is exported to script as a read-mostly view of the frame's state.
**Frozen**: shipped scripts read these names.

```text
CLASS render_device                       # one instance, via device()
  read-only: width, height, time_delta, f_time_delta,
             cam_pos, cam_dir, cam_top, cam_right,
             fov, aspect_ratio, precache_frame, frame
  time_global()  -> int                   # milliseconds, paused and scaled
  is_paused()    -> bool
  pause(on)                               # timers only; never sound

FREE FUNCTIONS
  device()             -> render_device
  app_ready()          -> bool            # the persistent layer finished loading
  time_global()        -> int
  time_global_async()  -> int             # the never-pausing clock; callable off the main thread
```

**Notes** — script gets `time_delta` (smoothed, paused) and `f_time_delta` (the same value as
a real number) but not the real measured delta, so script cannot accidentally integrate
against unpaused time. `time_global_async` is the deliberate exception, offered because
script sometimes needs to measure something across a pause.

The view and projection matrices are declared and commented out in the binding. Script can
read the camera basis but not the matrices; anything needing a projection must rebuild it
from the field of view and aspect.
