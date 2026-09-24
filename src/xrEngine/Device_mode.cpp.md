# src/xrEngine/Device_mode.cpp

> Display-mode enumeration and every window geometry decision: which monitor, which resolution, bordered or not, fullscreen exclusive or desktop-sized.

**Needs** — [`device.h`](device.h.md) · [`defines.h`](defines.h.md) · [`xr_input.h`](xr_input.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`Device_create.cpp`](Device_create.cpp.md)
**Tier floor** — T1: display enumeration and window-manager control are host services with no portable abstraction above them.

## Purpose

The engine exposes resolution, refresh rate, monitor and window style as four console variables that the options screen writes and the settings file persists. This file turns those four numbers into an actual window, and does the reverse for the options screen by publishing the host's mode list as named choices.

The central problem it solves: the saved settings may name a mode this display does not have. That happens constantly — the player changed monitor, the settings came from another machine, the desktop resolution changed. A mode that does not exist must be silently replaced by the nearest one that does, never rejected, because the alternative is a game that will not start.

## State

```text
RECORD DisplayMode            # four console variables, see defines.h
  monitor      : int          # index into the host's display list
  width        : int
  height       : int
  refresh_rate : int
  style        : one of {windowed, windowed_borderless, fullscreen_borderless, fullscreen}

# Enumerated once at startup and held until shutdown:
monitor_choices : list<(display_name, monitor_index)>
mode_choices    : map<monitor_index, list<(mode_label, mode_index)>>
# mode_label is "<width>x<height> (<rate>Hz)" and is the *key* a saved setting
# is matched against, so its exact spelling is load-bearing.
```

The four window styles are not two booleans. They are four genuinely different arrangements:

- **windowed** — a normal bordered, resizable window.
- **windowed borderless** — the same, without a frame; the engine supplies its own resize and drag behaviour through the hit test.
- **fullscreen borderless** — a borderless, non-resizable window sized to the *desktop's current* mode and stacked above everything. No mode switch, so alt-tab is instant. The requested width and height are ignored.
- **fullscreen** — a genuine exclusive mode switch to the requested width, height and refresh rate. The only style where refresh rate means anything.

## `fill_video_modes`

**Contract** — Enumerates every display the host reports, and for each one every display mode, building the two choice lists the console and options screen read. Also publishes each display's bounds, usable bounds and scale factor to the debug overlay so its floating windows can be placed on the right monitor. Fails hard if the host reports no displays — there is nothing sensible to do. Allocates strings that live until `cleanup_video_modes`.

```text
FUNCTION fill_video_modes()
  FAIL WITH "no display" IF display_count <= 0
  FOR EACH display index i
    monitor_choices.append("<i>. <display name>", i)
    FOR EACH mode index j of display i, from LAST to FIRST
      mode_choices[i].append("<w>x<h> (<rate>Hz)", j)
    publish display i's bounds, usable bounds and scale factor to the overlay
```

**Notes** — Modes are enumerated in reverse. The host lists them best-first (largest, highest refresh); reversing gives the options screen a list that reads small-to-large, which is what a player expects from a resolution dropdown.

**Notes** — The label, not the mode index, is what the saved setting is matched against. That makes the label format frozen against the user's settings file in practice: change the spacing and every saved fullscreen mode stops matching and falls back to the nearest. A rebuild should match on the numbers and keep the label for display only — the original conflates them.

**Notes** — Monitor scale factor is derived by dividing the host's reported dots-per-inch by 96. Ninety-six is the reference density the overlay toolkit defines as scale 1. The original flags that on one platform the host reports the panel's physical density rather than the user's chosen scaling, which makes the overlay the wrong size; the correct source there is the window system's own backing scale.

## `cleanup_video_modes`

**Contract** — Releases every enumerated label and empties both choice lists and the overlay's monitor list. Called at shutdown and before re-enumeration when displays are hot-plugged.

## `update_window_properties`

**Contract** — The single place the window's geometry is made to agree with the four settings. Resolves the requested resolution against what the display can do, moves the window to the requested monitor if it is not already there, sizes it, then applies the border/resizable/fullscreen combination for the requested style. Pumps the host's event queue so the changes take effect before the geometry is read back, then reads it back and republishes the display size to the debug overlay.

```text
FUNCTION update_window_properties()
  windowed = (style != fullscreen)
  select_resolution(windowed)

  IF the window is not on the requested monitor THEN
    leave fullscreen first                   # a mode switch is bound to one display
    move the window to that monitor's origin

  IF style != fullscreen_borderless THEN size the window to (width, height)
  ELSE                                  size the window to the desktop's current mode

  IF windowed THEN
    bordered  = (style == windowed)
    desktop_fs = ready AND style == fullscreen_borderless
    set bordered, set resizable = NOT desktop_fs, set desktop-fullscreen = desktop_fs
  ELSE IF ready THEN
    set not resizable, enter exclusive fullscreen
    set the window's display mode to (width, height, refresh_rate)

  pump host events
  update_window_rects()
  publish (width, height) and the framebuffer-to-client scale to the overlay
```

**Invariants** — Leaving fullscreen before changing monitor is not optional. An exclusive mode is owned by one display; moving a window that holds one to another display either fails or takes the mode with it.

**Notes** — Both fullscreen branches are guarded on the device being ready. During first bring-up the window is deliberately left windowed and hidden: entering fullscreen before a graphics device exists means a mode switch with nothing to present, which on some drivers produces a black screen the user cannot escape. Fullscreen is therefore reached on the *first reset* after creation, not at creation.

**Notes** — The framebuffer-to-client scale is the render resolution divided by the window's client size. It is not always one: on a high-density display the client size is in logical units and the framebuffer is larger, and the overlay needs the ratio to hit-test its widgets correctly.

## `update_window_rects`

**Contract** — Reads back the window's true geometry into two records: the client rectangle (origin at zero, the drawable size) and the bounds rectangle (screen position and size, expanded outward by the host's reported frame thickness). The bounds rectangle includes the frame because it is used for window dragging and for confining the pointer, both of which are in screen space.

## `select_resolution`

**Contract** — Decides the actual render resolution. Three cases in priority order, and the middle one is the interesting one.

```text
FUNCTION select_resolution(windowed)
  IF dedicated_server THEN
    width, height = 640, 480              # fixed; nothing is drawn
  ELSE IF width == 0 AND height == 0 AND refresh_rate == 0 THEN
    adopt the monitor's current desktop mode   # first run: no saved setting exists
  ELSE IF NOT windowed THEN
    # Only exclusive fullscreen must name a mode the display really has.
    IF the requested mode's label is not in this monitor's list THEN
      requested = nearest mode the host can offer for (w, h, rate)
      IF the host offers none THEN requested = the monitor's current desktop mode
      adopt requested
  render_width, render_height = width, height
```

**Notes** — An all-zero triple is the sentinel for "never configured", which is why zero is not a legal resolution rather than being validated away. First run therefore silently inherits the desktop, which is the behaviour a player expects and requires no first-run dialog.

**Notes** — The validity check runs only for exclusive fullscreen. A window may be any size, including larger than the display, and the borderless-fullscreen style takes the desktop size and ignores the request entirely. Validating those would refuse legitimate configurations.

## `set_window_draggable`

**Contract** — Enables engine-owned window dragging, which is only meaningful for a window that is both windowed and resizable; the request is silently downgraded otherwise. Also drops the window to 95% opacity while dragging is armed, as a visual cue that the window is in a move/resize mode rather than accepting game input.

## `on_error_dialog`

**Contract** — Called on both sides of a host message box raised while the game is running. Before: leave fullscreen and release input capture, because a modal dialog behind an exclusive-fullscreen window is invisible and a captured pointer cannot reach it. After: restore the window properties and re-acquire capture, but only if capture was exclusive to begin with.

## `on_fatal_error`

**Contract** — Gets the window out of the way unconditionally before the process dies: leave fullscreen, drop always-on-top, then show, minimize and hide it. Nothing else may run after this, so it does not care about consistency — only that no exclusive-fullscreen window is left covering the crash report.

**Notes** — The show-then-minimize-then-hide sequence is a shove rather than a decision: some window managers ignore a minimize on a hidden window and ignore a hide on a fullscreen one, so each step is issued in the order that makes the next one legal. A rebuild needs the same outcome and may reach it differently.

## `application_window`

**Contract** — Reports the engine's window to code that needs it through the windowing abstraction rather than as a native handle.
