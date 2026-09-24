# src/Include/editor/engine.hpp

> What the editor library may ask of the engine host: pump the frame, own the window's messages, and read and write the weather being edited.

**Needs** — [`ide.hpp`](ide.hpp.md) · [`property_holder_base.hpp`](property_holder_base.hpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`ide.hpp`](ide.hpp.md) · [`interfaces.hpp`](interfaces.hpp.md) · [`property_holder_base.hpp`](property_holder_base.hpp.md) · [`engine_include.hpp`](../../editors/xrWeatherEditor/engine_include.hpp.md) · [`entry_point.cpp`](../../editors/xrWeatherEditor/entry_point.cpp.md) · [`ide_impl.cpp`](../../editors/xrWeatherEditor/ide_impl.cpp.md) · [`property_holder.cpp`](../../editors/xrWeatherEditor/property_holder.cpp.md) · [`property_holder.hpp`](../../editors/xrWeatherEditor/property_holder.hpp.md) · [`property_string_shared_str.cpp`](../../editors/xrWeatherEditor/property_string_shared_str.cpp.md) · [`property_string_shared_str.hpp`](../../editors/xrWeatherEditor/property_string_shared_str.hpp.md) · [`window_ide.cpp`](../../editors/xrWeatherEditor/window_ide.cpp.md) · [`window_ide.h`](../../editors/xrWeatherEditor/window_ide.h.md) · [`window_view.cpp`](../../editors/xrWeatherEditor/window_view.cpp.md) · [`window_view.h`](../../editors/xrWeatherEditor/window_view.h.md) · _and 4 more_
**Tier floor** — T1: it passes raw platform window messages through by their four native arguments, and returns a native result.

## Purpose

The weather editor's engine host runs a real level with a real renderer, but it does not own its own window or message loop — the editor library's window framework does. This interface is the host's side of that inversion: the editor drives, the engine responds. Everything an editor window needs to do to a running engine is here, and nothing else is.

Its second half is a complete accessor surface for the weather model, because editing weather is what this editor is for.

## State

The interface itself holds nothing; it is the host engine's facade. What it exposes is a view onto:

```text
RECORD WeatherEditingState          # conceptually, on the host side
  environment        : Environment      # the running weather system
  current_weather    : text             # which cycle is being edited
  current_frame      : text             # which keyframe within it
  frame_track_time   : real             # the scrub position within a keyframe blend
  weather_track_time : real             # the scrub position along the day
  paused             : bool
  time_factor        : real             # how fast game time runs while editing
```

**Invariants** — the current keyframe always belongs to the current cycle; setting the cycle resets the keyframe. The three property views described below always describe the current keyframe, the blend result, and the keyframe being blended towards — in that order and never any other.

## `engine_base` — what the host must provide

### Frame and window

```text
FUNCTION on_message(window, message, arg_a, arg_b, out result) -> handled : bool
FUNCTION on_idle()
FUNCTION on_resize()
FUNCTION pause(value : bool)
FUNCTION capture_input(value : bool)
FUNCTION disconnect()
FUNCTION quit_requested() -> bool
```

**Contract** — `on_message` is offered every window message the editor's window receives, before the window framework handles it, and returns whether it consumed it. `on_idle` advances the engine by one frame; the editor's idle handler calls it in a tight loop until a message arrives, which is how the engine runs at full rate inside an event-driven application.

`capture_input` decides whether mouse and keyboard go to the engine's camera or to the editor's widgets — the editor turns it on when the pointer is over the three-dimensional view. `disconnect` unloads the level. `quit_requested` lets the engine end the session on its own (a console quit command), which the editor polls in its idle loop.

**Notes** — Message interception plus an idle pump is the *only* way to run a fixed-rate engine inside a retained-mode window framework, and a rebuild will meet the same problem whatever framework it uses. The shape to keep is: **the engine exposes one "advance a frame" call and one "would you like this input event" call, and owns no loop of its own.** Everything platform-specific about the message quadruple is incidental; what matters is that the engine gets first refusal on input and that its frame is a callable rather than a loop.

### String interning

```text
FUNCTION intern(value : text, out result : SharedString)
FUNCTION text_of(value : SharedString) -> text
```

**Contract** — convert between plain text and the engine's interned string type, in both directions. The editor library cannot construct an interned string itself — the intern table lives in the host and is not safely reachable across the boundary — so the host does it.

**Notes** — This is a pure consequence of the split: two modules, one intern table, and only one of them may touch it. A rebuild in one language deletes both methods. The decision worth recording is that **the engine's interned string is not a type that may cross the module boundary by value**, which is also why the property grid below takes plain text everywhere except where it writes a stored value.

### The weather model

```text
FUNCTION environment() -> Environment          # the live weather system

FUNCTION weather() -> text                     # the cycle being edited
FUNCTION set_weather(name : text)
FUNCTION current_frame() -> text               # the keyframe within it
FUNCTION set_current_frame(id : text)

FUNCTION frame_track() -> real                 # scrub position within the current blend
FUNCTION set_frame_track(time : real)
FUNCTION weather_track() -> real               # scrub position along the day
FUNCTION set_weather_track(time : real)

FUNCTION weather_paused() -> bool
FUNCTION set_weather_paused(value : bool)
FUNCTION weather_time_factor() -> real
FUNCTION set_weather_time_factor(value : real)

FUNCTION weather_current_time() -> text        # as an authored time-of-day string
FUNCTION set_weather_current_time(value : text)
```

**Contract** — reading and writing these drives the live weather immediately; the editor's three-dimensional view shows the result on the next idle frame. The two track positions are the timeline scrubbers: one moves within the blend between two adjacent keyframes, the other moves the whole day.

Time of day crosses as **authored text, not a number**, because that is the form the configuration files store and the form the editor's field displays; converting it twice would introduce rounding the author would see.

### Property views

```text
FUNCTION current_frame_properties() -> PropertyHolder
FUNCTION blend_frame_properties()   -> PropertyHolder
FUNCTION target_frame_properties()  -> PropertyHolder
```

**Contract** — three [property holders](property_holder_base.hpp.md) the editor binds to three grids: the keyframe at the start of the current blend, the interpolated result at the current instant, and the keyframe being blended towards. Each is populated by the host with the keyframe's fields bound to live getters and setters, so editing a cell changes the running weather at once.

**Notes** — Showing the *interpolated* keyframe as an editable-looking grid is the design decision here, and it is what makes this editor usable: an author sees not just the two keyframes they wrote but the value the engine actually uses at this instant, in the same units, beside them. Its fields are read-only in effect.

### Persistence and clipboard

```text
FUNCTION save_weathers()
FUNCTION reload_current_time_frame()
FUNCTION reload_target_time_frame()
FUNCTION reload_current_weather()
FUNCTION reload_weathers()

FUNCTION copy_time_frame(out buffer, buffer_size) -> ok : bool
FUNCTION paste_current_time_frame(buffer, size) -> ok : bool
FUNCTION paste_target_time_frame(buffer, size) -> ok : bool
FUNCTION add_time_frame(buffer, size) -> ok : bool
```

**Contract** — the four reload calls are the undo mechanism: there is no undo stack, so reverting means re-reading the configuration from disk at one of four granularities — this keyframe, the target keyframe, this whole cycle, or every cycle. Saving writes the whole weather set back out in the configuration format it was read from.

Copy and paste move a keyframe through a caller-supplied text buffer in that same configuration format. `copy` fails rather than truncating if the buffer is too small; `add` creates a new keyframe from a pasted block rather than overwriting one.

**Notes** — Serializing a keyframe to its own on-disk text form for the clipboard is a good decision for a bad reason: the two modules cannot share a structured type cheaply, so text was the available common currency. The side effect is that an author can paste a keyframe into a text editor and back, and can paste one between two running editors. A rebuild should keep the text form for exactly that reason.

The absence of a real undo stack is the interface's most significant limitation and it is structural — the host owns the data, the editor owns the commands, and neither has a history. A rebuild should put the history on the host side, where the mutations happen.
