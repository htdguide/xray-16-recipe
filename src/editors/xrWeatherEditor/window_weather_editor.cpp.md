# src/editors/xrWeatherEditor/window_weather_editor.cpp

> The timeline: scrub the day, watch the blend, and edit the two keyframes it runs between — with every control kept in step with an engine that is also moving.

**Needs** — [`window_weather_editor.h`](window_weather_editor.h.md) · [`window_ide.h`](window_ide.h.md) · [`window_view.h`](window_view.h.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md)
**Used by** — [`window_weather_editor.h`](window_weather_editor.h.md)
**Tier floor** — T2: it holds a native engine reference and owns four bound callables into it.

## Purpose

The tool's centre and its hardest problem: **a dozen controls and a running engine that
are all views of the same clock, each able to change it.** Almost every line here is about
a value changing in two directions at once without oscillating.

## State

See [`window_weather_editor.h`](window_weather_editor.h.md).

## The re-entry discipline

Every two-way control in this panel obeys the same rule, and it is the single most
important pattern in the file:

```text
# when the code writes a control
guard = true
control.value = new value          # the control reports a change; the handler sees the guard
guard = false

# when the control reports a change
ON control_changed
  IF guard THEN RETURN             # it was us, not the author
  ... act on the author's edit
```

**Invariants** — each guard is asserted clear before it is set, so a nested write is a
detected error rather than a silent skip. There are four of them: the keyframe selector,
the frame scrubber, the clock and the day scrubber.

**Notes** — This is the cost of a model that changes on its own. The engine advances time
every frame; the panel must show it; showing it writes the controls; writing a control
looks exactly like the author moving it. A rebuild with a change-origin on its events —
"this came from the model, not the user" — deletes all four guards, and should.

The fifth flag is different in kind: `updates_enabled` is cleared while the keyframe combo
box is *dropped down*, so the per-frame refresh does not change the selection out from
under an open list.

## `bind_weather_editor` and `refill`

```text
FUNCTION bind_weather_editor(cycle_names, cycle_count, frame_names_of, frame_count_of)
  REQUIRE none is already bound
  keep all four
  refill()

FUNCTION refill()
  time_factor.value = engine.time_factor
  cycle_combo.items = every name FROM cycle_names()
  index = position of engine.current_cycle IN cycle_combo.items
  IF index NOT found AND the list is not empty THEN index = 0
  cycle_combo.selected = index
  frame_combo.items = empty
  IF index NOT found THEN RETURN
  fill_frames(engine.current_cycle)

FUNCTION fill_frames(cycle : text)
  frame_combo.items = every name FROM frame_names_of(cycle)
  index = position of engine.current_frame IN frame_combo.items
  IF index NOT found AND the list is not empty THEN index = 0
  GUARDED: frame_combo.selected = index
```

**Contract** — rebuilds both selectors from the engine's current state. Falls back to the
first entry when the engine's selection is not in the list, which happens after a reload
renames or removes it.

**Invariants** — the data sources are bound exactly once, from the engine side; see
[`editor_environment_weathers_manager.cpp`](../xrWeatherEngine/editor_environment_weathers_manager.cpp.md).
The keyframe names must be fetched before the count, an ordering the engine side documents
as a hazard — this panel happens to satisfy it.

**Notes** — Rebuilding on every focus into the panel, as well as after a reload, is how the
selectors stay honest without a notification protocol. Pull, not push, all the way down.

## `on_load_finished`

```text
FUNCTION on_load_finished()
  refill()
  blend.selected_object = engine.blend_frame_properties()      # attached once, forever
  toggle pause
  load()                        # the three grids' persisted state
  load_finished = true
  act as if the frame scrubber had moved       # forces a first refresh
```

**Contract** — runs once, when the level has loaded.

**Notes** — **The blend grid is attached once and never re-attached**, because the
interpolated keyframe is a single object that lives for the session; the other two grids
are re-attached whenever the blend moves to a different pair. That asymmetry is the direct
consequence of
[`editor_environment_manager.cpp`](../xrWeatherEngine/editor_environment_manager.cpp.md)
creating exactly one interpolation target.

## `update_frame` — the per-frame refresh

```text
FUNCTION update_frame()
  GUARDED: clock.text = engine.current_time_of_day

  GUARDED: IF day_track.value != round(1000 * engine.day_position)
             day_track.value = round(1000 * engine.day_position)

  GUARDED: IF frame_track.value != round(1000 * engine.frame_position)
             frame_track.value = round(1000 * engine.frame_position)

  properties = engine.current_frame_properties()
  IF properties EXISTS AND current.selected_object IS NOT properties
    current.selected_object = properties
    blend.refresh()

  IF NOT engine.weather_paused AND load_finished
    blend.refresh()

  properties = engine.target_frame_properties()
  IF properties EXISTS AND target.selected_object IS NOT properties
    target.selected_object = properties
    blend.refresh()

  IF NOT updates_enabled THEN RETURN          # the keyframe list is dropped down
  name = engine.current_frame
  IF name == frame_combo.selected THEN RETURN
  GUARDED: frame_combo.selected = position of name
```

**Contract** — brings every control into line with the engine, once per frame. Each write
is guarded and each is skipped when the value has not changed.

**Invariants** — both scrubbers are integers over a thousand steps, so the engine's
zero-to-one position is scaled by a thousand and rounded. The comparison before writing is
what keeps the rounding from producing a visible jitter.

**Notes** — A thousand steps is the resolution of the whole tool's timeline, and it is
where the numbers come from: across a day that is 86.4 seconds of game time per step, and
within a blend it is a thousandth of the interval. Fine enough that scrubbing feels
continuous, coarse enough that the scrubber's own integer value is stable frame to frame.

**The blend grid is refreshed on three different conditions** — either keyframe changed,
or time is running — and not on a fourth, which is when the author is dragging a scrubber;
that case is handled separately in `on_idle`. A rebuild can refresh it unconditionally and
lose only performance.

## `on_idle`

```text
FUNCTION on_idle()
  update_frame()
  IF pointer_down THEN blend.refresh()
  IF the focused control IS current THEN view.property_grid = current
  ELSE IF the focused control IS target THEN view.property_grid = target
```

**Notes** — Two things per frame that `update_frame` does not do. Refreshing while a
scrubber is dragged is what makes scrubbing show its effect in the middle grid live.
Checking which grid has focus, every frame, is how the view's drag-and-sample gestures know
which of *three* grids to act on — the one-grid panels do it on focus loss instead, and
this one cannot, because moving between its own two grids is not a focus loss for the
panel.

## Selecting a cycle and a keyframe

```text
ON cycle_selected
  IF nothing selected THEN RETURN
  engine.current_cycle = the selected name
  cycle_combo.selected = position of engine.current_cycle    # the engine may have refused
  fill_frames(engine.current_cycle)

ON frame_selected
  IF guarded THEN RETURN
  IF nothing selected THEN RETURN
  WITH weather running FOR THE DURATION:
    engine.current_frame = the selected name
    update_frame()

ON previous_clicked : step frame_combo back one, wrapping to the last
ON next_clicked     : step frame_combo on one, wrapping to the first
```

**Contract** — selecting a cycle asks the engine and then **re-reads what the engine
actually chose**, because the engine silently refuses an unknown name. Selecting a keyframe
moves the blend to start there.

**Notes** — Re-reading after writing is the correct handling of a setter that can decline,
and it appears only here; every other control trusts its write. The cycle setter is the
only one that can decline — see
[`engine_impl.cpp`](../xrWeatherEngine/engine_impl.cpp.md).

Wrapping the step buttons at both ends makes the keyframe list a loop, which is what it
is: the last keyframe of a day blends into the first.

## Running the weather while a control acts

```text
# a scoped guard used by four handlers
WITH weather running FOR THE DURATION:
  remember whether the weather was paused
  unpause
  ... do the thing ...
  restore the remembered pause state
```

**Contract** — four handlers — keyframe selection, both scrubbers, and the clock — run the
weather while they act and then put the pause state back.

**Notes** — This is the panel's second-most important pattern and it is not obvious. The
engine's time-setting path only takes effect while the weather is *running*; setting time
on a paused weather leaves the blend stale. So an author scrubbing a paused day must have
the weather briefly unpaused underneath them, without ever seeing it run. The
[engine side](../xrWeatherEngine/engine_impl.cpp.md) has its own version of the same dance
for the same reason.

**A rebuild should fix this at the engine instead**: a "set the time to exactly this and
recompute" operation that works regardless of pause makes all five copies of this guard
disappear.

## The two scrubbers

```text
ON frame_track_changed
  IF guarded THEN RETURN
  WITH weather running FOR THE DURATION:
    engine.frame_position = frame_track.value / 1000
    update_frame()
  IF the change came from the author THEN engine.on_idle()      # draw it now

ON day_track_changed
  IF guarded THEN RETURN
  WITH weather running FOR THE DURATION:
    engine.day_position = day_track.value / 1000
    update_frame()
  engine.on_idle()

ON either_track_pointer_down(at x)
  engine.weather_paused = true
  margin = 12
  span = (maximum - minimum) / (control width - 2 * margin)
  value = clamp((x - margin) * span + minimum, minimum, maximum)
  track.value = value                 # jump to the click, do not step toward it
  pointer_down = true

ON either_track_pointer_up
  engine.weather_paused = the pause button's state
  blend.refresh()
  pointer_down = false
```

**Contract** — dragging a scrubber moves the clock and draws the result immediately.
Clicking anywhere on a scrubber jumps straight there. Pressing pauses the weather;
releasing restores whatever the pause button says.

**Invariants** — the twelve-pixel margin is the width of the scrubber's own thumb at each
end; the usable track is the control's width less two of them. Without it, a click at the
far right lands short of the maximum.

**Notes** — **Jump-to-click rather than page-toward-click** is the right behaviour for a
timeline and is not what the widget does by default, which is why the position has to be
computed by hand. The margin is the one magic number here and it is a property of the
widget's own drawing, not of the model; a rebuild measures its own thumb.

Pausing on press and restoring on release means a drag is always against a still clock,
so the author's position is not fighting time's advance. That is why `pointer_down` also
drives the extra refresh in `on_idle`: paused, nothing else would redraw the blend.

Only the frame scrubber tests whether the change came from the author before forcing a
frame; the day scrubber always forces one. An inconsistency in the original, not a
distinction.

## The clock

```text
ON clock_text_changed
  IF guarded THEN RETURN
  IF the text still contains a mask placeholder THEN RETURN    # half-typed
  WITH weather running FOR THE DURATION:
    engine.current_time_of_day = the text
    update_frame()
  blend.refresh()
```

**Contract** — typing a complete time jumps the day to it. An incomplete entry is ignored.

**Notes** — The field is masked to `00:00:00`, and the placeholder test is what makes
typing into it usable: without it, every keystroke would submit a partial time and the day
would lurch. **A rebuild needs the same property — act on a complete value, not on every
keystroke** — whatever its input widget offers.

The engine's parser does not validate, so `99:99:99` is accepted by the mask and produces a
time past the end of the day; see
[`engine_impl.cpp`](../xrWeatherEngine/engine_impl.cpp.md).

## Pause and time factor

```text
ON pause_clicked
  pause.image = pause.image XOR 1        # the image index IS the state
  engine.weather_paused = (pause.image != 0)

ON time_factor_changed
  engine.time_factor = time_factor.value
```

**Notes** — **The pause button's image index is its state**, with no separate flag, which
is why two other handlers read `pause.image` to restore the pause state after a drag.
Storing state in a widget's appearance is the kind of thing a rebuild should not copy —
but note that it is *read back* in two places, so a rebuild must introduce the flag those
two need.

The time factor spinner runs from a tenth to a hundred thousand in steps of one with one
decimal place shown, against an engine that clamps to a hundredth at the bottom — so the
spinner's floor is ten times the engine's. Deliberate or not, the spinner's is the more
usable.

## The clipboard and the four grid commands

```text
ON copy_clicked
  buffer of 4096 bytes
  IF NOT engine.copy_time_frame(INTO buffer) THEN RETURN      # too large: fail silently
  clipboard.text = buffer

ON paste_current_clicked
  engine.paste_current_time_frame(FROM clipboard.text)
  fill_frames(engine.current_cycle)
  current.refresh()

ON paste_target_clicked
  ... the same, against the target grid

ON create_from_clicked
  buffer of 4096 bytes
  IF NOT engine.copy_time_frame(INTO buffer) THEN RETURN
  engine.add_time_frame(FROM buffer)
  fill_frames(engine.current_cycle)
  refresh all three grids
  frame_track.value = 1000                  # jump to the end of the blend

ON reload_current_clicked : engine.reload_current_time_frame(); current.refresh()
ON reload_target_clicked  : engine.reload_target_time_frame();  target.refresh()
```

**Contract** — copy serialises the interpolated keyframe to the clipboard as configuration
text; the two pastes overwrite a keyframe's appearance without moving it; create-from makes
a *new* keyframe at the current instant; the two reloads revert one keyframe from disk.

**Invariants** — the transfer buffer is four kilobytes and a keyframe that serialises
larger is silently dropped. A full keyframe section is roughly a quarter of that, so the
margin is wide — but the failure is invisible, and a rebuild should size the buffer to the
content or report the failure.

**Notes** — **Create-from is the tool's most-used command and it composes two others**:
serialise the blend, then add it as a keyframe. Because the blend's identifier is the
current clock time — see
[`editor_environment_weathers_time.cpp`](../xrWeatherEngine/editor_environment_weathers_time.cpp.md) —
the new keyframe lands exactly where the author was looking, with exactly what they were
seeing. Nothing else has to be passed.

Moving the frame scrubber to its end afterwards is the small finishing touch: the new
keyframe is now the *target* of the blend the author was in, and the end of that blend is
where the new keyframe's own values apply. Without it the view would jump.

Every paste rebuilds the keyframe selector, because a paste can change an identifier and
`add` certainly does.

## Focus and per-grid state

```text
ON current_focus_gained OR current_focus_lost : view.property_grid = current
ON target_focus_gained  OR target_focus_lost  : view.property_grid = target

FUNCTION save(root)  : write each of the three grids' state under its own key
FUNCTION load(root)  : read them back, if present
```

**Notes** — This panel latches the view's target grid on focus *both* ways, unlike the
single-grid panels which latch only on the way out. It must: moving from the start grid to
the target grid is not a focus change the panel sees, so the entering grid has to claim the
view itself.

The three grids' own state — which groups are expanded, which row is selected — is
persisted per grid, keyed separately, so the author reopens the tool with the same rows
open. It is saved from
[`window_ide_serialize.cpp`](window_ide_serialize.cpp.md) and restored in
`on_load_finished`, not at construction, because the grids have no content until the level
has loaded.
