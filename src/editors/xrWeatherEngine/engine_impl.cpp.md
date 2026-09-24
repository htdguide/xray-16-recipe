# src/editors/xrWeatherEngine/engine_impl.cpp

> The engine's answers to the editor: advance one frame, hand over input, and read and write the weather that is running right now.

**Needs** — [`engine_impl.hpp`](engine_impl.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`xrEngine/IGame_Level.h`](../../xrEngine/IGame_Level.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`engine_impl.hpp`](engine_impl.hpp.md)
**Tier floor** — T1: it pumps the platform event queue, forwards native window messages, and hands the device an input claim.

## Purpose

The hinge of the whole editor. On one side is an event-driven application that owns the
window and the loop; on the other is a fixed-rate engine that expects to own both. This
file is the adapter, and almost every decision in it is about the two disagreeing.

## State

```text
RECORD EngineHost
  input_receiver  : InputReceiver
  input_captured  : bool          # invariant: matches whether input_receiver holds the claim
```

Construction creates the input claim but does not take it; teardown drops the claim before
releasing it, so the device is never left pointing at a dead receiver.

## Frame and window

```text
FUNCTION on_message(window, message, arg_a, arg_b, out result) -> handled : bool
  RETURN device.on_message(window, message, arg_a, arg_b, out result)

FUNCTION on_idle()
  pump_platform_events()
  device.process_frame()

FUNCTION on_resize()
  IF console EXISTS
    console.execute("vid_restart")
```

**Contract** — `on_message` is pure forwarding: the device gets first refusal on every
message the editor's window receives. `on_idle` advances the engine by exactly one frame,
and is called from the editor's paint handler, not from a loop of its own.

**Notes** — Pumping the platform event queue *inside* the frame call is the load-bearing
part, and it looks redundant beside an application that already has a message loop: the
editor's loop delivers messages to widgets, but the windowing layer the engine reads input
through keeps its own queue, and nothing else drains it. **Two event queues, one message
source, and the engine must drain its own** — a rebuild that shares one queue between the
application and the engine deletes this line and must then re-route the engine's input.

Resizing restarts the graphics device rather than resizing the swap chain, through the
console's own command. That is heavy — a full device teardown on every drag of a window
edge — and it is the honest reading of what the engine offers: its display state is
reconfigured by command, not by API, and the editor has no cheaper lever.

## `pause`, `capture_input`, `disconnect`, `quit_requested`

```text
FUNCTION pause(value : bool)
  IF value == device.paused
    RETURN
  device.pause(value, also_sound = true, also_time = true, reason = "editor query")

FUNCTION capture_input(value : bool)
  IF value == input_captured
    RETURN
  input_captured = value
  IF value THEN input_receiver.capture() ELSE input_receiver.release()

FUNCTION disconnect()
  console.execute("quit")

FUNCTION quit_requested() -> bool
  RETURN platform_quit_was_requested()
```

**Contract** — both setters are idempotent and return early on no change; that is not a
micro-optimisation, it is required, because taking an input claim twice and releasing it
once leaves the device holding a claim nobody will release.

**Notes** — Pausing suspends sound and game time together, and names its reason, because
the engine's pause is refcounted by reason across subsystems. Closing the editor's window
does not close the editor: the window's close handler cancels the close, saves the
session, and asks the engine to quit; the engine then reports a quit request on its next
poll and the application ends from that side. **The engine, not the window, decides when
the process ends** — which is what lets a console `quit` typed into the three-dimensional
view close the editor cleanly.

## String interning

```text
FUNCTION intern(value : text, out result : SharedString)
  result = value

FUNCTION text_of(value : SharedString) -> text
  RETURN value.text
```

**Notes** — Two one-line methods that exist only because the intern table lives in this
module and the editor cannot reach it. A rebuild with one string type deletes them. See
[`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md).

## `environment`

**Contract** — constructs and returns the editor's weather system —
[`editor_environment_manager`](editor_environment_manager.cpp.md) — which is the ordinary
run-time weather system with every authored record replaced by an editable one. The engine
calls this once during start-up instead of building its own.

**Notes** — This single substitution is the whole mechanism by which the editor edits
*the running world* rather than a copy of it. There is no separate document model: the
engine's live weather objects **are** the document.

## Selecting the weather cycle

```text
FUNCTION set_weather(name : text)
  IF no game session
    RETURN
  IF name == current cycle name
    RETURN
  IF name IS NOT a known cycle
    RETURN                                   # silently: the editor may ask before load
  now = environment.game_time
  environment.set_weather(name, forced = true)
  level.set_environment_time_factor(floor(now), environment.time_factor)
  environment.select_envs(now)
```

**Contract** — switches the cycle being edited and re-selects the keyframe pair for the
current instant, so the view updates without waiting for time to advance. Unknown names
are ignored rather than reported; the editor's cycle list is built from the same data, so
an unknown name means the editor asked before the model was loaded.

## Selecting a keyframe, and the wrap at midnight

```text
FUNCTION set_current_frame(id : text)
  FOR EACH (frame, next) IN adjacent_pairs_cyclic(current_cycle.frames)
    IF frame.id != id
      CONTINUE
    environment.current[0] = frame
    environment.current[1] = next          # wraps: the last frame's target is the first
    set_time = true
    IF frame.exec_time < next.exec_time
      # an ordinary interval inside one day
      IF environment.game_time > frame.exec_time
        AND environment.game_time < next.exec_time
        set_time = false                   # already inside it; do not jump
    ELSE
      # the interval that crosses midnight
      IF environment.game_time > frame.exec_time
        set_time = false
    IF set_time
      environment.set_game_time(frame.exec_time, environment.time_factor)
      level.set_environment_time_factor(floor(frame.exec_time * 1000), environment.time_factor)
    BREAK
```

**Contract** — makes a keyframe the start of the current blend and the next one its
target. Jumps game time to the keyframe's own time *unless* the clock already sits inside
that keyframe's interval, in which case the author's scrub position is preserved.

**Invariants** — a cycle's keyframes are sorted by time of day and the list is cyclic:
the last keyframe blends into the first, across midnight. Every time calculation in this
file exists to handle that one wrap.

**Notes** — "Do not jump if we are already inside the interval" is the decision that makes
keyframe selection feel like selection rather than a seek. Note the asymmetry between the
two branches: for an ordinary interval the clock must be inside both bounds, but for the
midnight-crossing interval only the lower bound is tested — because past the lower bound
there is nothing above it before the day ends.

## The two timeline positions

```text
FUNCTION set_frame_track(t : real)          # t in [0,1] within the current blend
  REQUIRE 0 <= t <= 1
  start = current[0].exec_time
  stop  = current[1].exec_time
  IF start < stop
    now = start + t * (stop - start)
  ELSE                                      # crosses midnight
    stop = stop + SECONDS_PER_DAY
    now = start + t * (stop - start)
    IF now >= SECONDS_PER_DAY
      now = now - SECONDS_PER_DAY
  environment.set_game_time(now, environment.time_factor)
  level.set_environment_time_factor(floor(now * 1000), environment.time_factor)

FUNCTION frame_track() -> real
  start = current[0].exec_time
  stop  = current[1].exec_time
  now   = environment.game_time
  IF start >= stop                          # crosses midnight: unroll onto one line
    IF now >= start THEN clamp(now, start, SECONDS_PER_DAY)
    ELSE clamp(now, 0, stop)
    IF now <= stop
      now = now + SECONDS_PER_DAY
    stop = stop + SECONDS_PER_DAY
  ELSE
    clamp(now, start, stop)
  RETURN (now - start) / (stop - start)

FUNCTION set_weather_track(t : real)        # t in [0,1) across the whole day
  was_paused = environment.paused
  environment.paused = false
  environment.set_game_time(t * SECONDS_PER_DAY, environment.time_factor)
  environment.paused = true
  environment.set_game_time(t * SECONDS_PER_DAY, environment.time_factor)
  environment.paused = was_paused
  environment.invalidate()
  environment.lerp()

FUNCTION weather_track() -> real
  RETURN environment.game_time / SECONDS_PER_DAY
```

**Contract** — the frame track is a position inside the current blend; the weather track
is a position in the day. Both write game time directly and take effect on the next frame.

**Invariants** — after unrolling, the interval start is strictly less than its end; the
day is 86400 seconds and a keyframe's time is a count of seconds from midnight.

**Notes** — Two things here look arbitrary and are not.

First, **a day that wraps is handled by unrolling, not by modular arithmetic**: the
midnight-crossing interval is converted into an ordinary interval by adding a day to its
end, the position is computed on that line, and the result is folded back. Every
time-of-day computation in this module repeats that pattern —
[`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md) does it
again for the blend itself — and a rebuild should factor it out once.

Second, **setting the weather track writes the time three times around a pause toggle.**
The engine's clock is advanced by the game level, and setting it while unpaused lets the
level's own advance overwrite the value before the next frame reads it; setting it again
while paused pins it. The final restore puts the author's pause state back. This is a
symptom of the clock having two owners, and a rebuild that gives the editor an authoritative
"set the time to exactly this" operation on the environment deletes the dance. Note that
the frame track, which sets time the same way, does *not* do it — an inconsistency in the
original rather than a distinction.

The multiplication by 1000 when telling the level about the time factor is a unit change:
the environment keeps time of day in seconds, the level keeps it in milliseconds.

## The three property views

```text
FUNCTION current_frame_properties() -> optional<PropertyHolder>
  IF environment.current[0] IS none THEN RETURN none
  RETURN as_editable_keyframe(environment.current[0]).property_holder()

FUNCTION blend_frame_properties() -> optional<PropertyHolder>
  IF environment.current_env IS none THEN RETURN none
  RETURN as_editable_keyframe(environment.current_env).property_holder()

FUNCTION target_frame_properties() -> optional<PropertyHolder>
  IF environment.current[1] IS none THEN RETURN none
  RETURN as_editable_keyframe(environment.current[1]).property_holder()
```

**Notes** — All three reinterpret a run-time keyframe as an editable one without checking.
That is sound *only* because the editor's weather system built every keyframe in the
process, including the interpolation target, which
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) creates explicitly
for this purpose. **In the editor, there is no such thing as a non-editable keyframe.**

## Pause and time factor

```text
FUNCTION set_weather_paused(value : bool)
  environment.paused = value

FUNCTION set_weather_time_factor(value : real)
  clamp(value, 0.01, 100000)
  IF level AND session EXIST
    level.set_environment_time_factor(floor(environment.game_time * 1000), value)
  IF session EXISTS
    environment.time_factor = value
```

**Contract** — the time factor is how many game seconds pass per real second while
editing. Clamped to a hundredth at the low end and a hundred thousand at the high: below
the floor the day effectively stops and the author loses the ability to unstick it from
the dialog, and the ceiling is where a day passes in under a second and the blend is no
longer inspectable. Reading it with no session returns one.

## Time of day as text

```text
FUNCTION weather_current_time() -> text
  RETURN environment.current_env.identifier      # already formatted HH:MM:SS

FUNCTION set_weather_current_time(value : text)
  parse value AS hours ":" minutes ":" seconds
  was_paused = environment.paused
  environment.paused = false
  environment.set_game_time(hours*3600 + minutes*60 + seconds, environment.time_factor)
  environment.paused = was_paused
  environment.invalidate()
  environment.lerp()
```

**Notes** — The interpolated keyframe carries the current clock time *as its identifier* —
see [`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md),
which rewrites it on every blend. That is why reading the time of day is a field read and
not a format call: the formatting already happened where the blend happened. Parsing does
not validate; a malformed string yields whatever the parse left behind. The editor's field
is masked to `00:00:00` so malformed text does not normally reach here.

## Persistence, reload and clipboard

```text
FUNCTION save_weathers()
  as_editor_environment(environment).save()

FUNCTION copy_time_frame(out buffer, size) -> ok : bool
  RETURN weathers.save_current_blend(buffer, size)

FUNCTION paste_current_time_frame(buffer, size) -> ok : bool
FUNCTION paste_target_time_frame(buffer, size) -> ok : bool
FUNCTION add_time_frame(buffer, size) -> ok : bool
  # each delegates to the weathers manager

FUNCTION reload_current_time_frame()
FUNCTION reload_target_time_frame()
  # each delegates; no time fixup needed, the keyframe list did not change

FUNCTION reload_current_weather()          # and reload_weathers(), identically
  now = environment.game_time
  weathers.reload_current_weather()        # or weathers.reload()
  level.set_environment_time_factor(floor(now), environment.time_factor)
  environment.current[0] = none
  environment.current[1] = none
  environment.select_envs(now)
  REQUIRE environment.current[1] EXISTS
  IF environment.current[1].exec_time == now
    environment.select_envs(now + 0.1)     # nudge off an exact boundary
```

**Contract** — the two cycle-level reloads discard the in-memory keyframes and re-read
them from disk, then rebuild the current blend around the preserved clock time. The two
keyframe-level reloads re-read one keyframe in place and need no fixup.

**Notes** — Two details carry weight.

Clearing both ends of the blend before re-selecting is required, not tidy: the old
keyframe objects were destroyed by the reload, and selection would otherwise compare
against freed records.

The nudge past an exact boundary is the interesting one. Landing with the clock exactly on
the *target* keyframe's time leaves a blend of zero length, and the position within it is
then undefined — a division by zero in `frame_track`. Advancing a tenth of a second
selects the next pair instead. **The blend interval is half-open; an author who reloads at
exactly a keyframe's time gets the interval that starts there, never the one that ends
there.**

The time factor is restored using `floor(now)` here but `floor(now * 1000)` elsewhere in
this same file — seconds where milliseconds are expected. This looks like a defect rather
than a decision, and a rebuild should use one unit throughout.
