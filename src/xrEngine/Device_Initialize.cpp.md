# src/xrEngine/Device_Initialize.cpp

> Brings the window into existence before anything else — title, icon, size limits, drag-and-resize behaviour — and starts the two clocks the whole engine reads.

**Needs** — [`device.h`](device.h.md) · [`xr_input.h`](xr_input.h.md) · [`Render.h`](Render.h.md) · [`GameFont.h`](GameFont.h.md) · [`PerformanceAlert.hpp`](PerformanceAlert.hpp.md) · [`embedded_resources_management.h`](embedded_resources_management.h.md) · [`editor_base.h`](editor_base.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it creates an OS window, extracts a platform icon resource from the executable image, and hands back a native window handle for the graphics backend to bind a swap chain to.

## Purpose

This is the first thing the device does and it happens before any graphics backend exists. It must, because the backend needs a window to attach to, and because a fatal error before this point has nowhere to display a message. Window creation is therefore separated from graphics-device creation ([`Device_create.cpp`](Device_create.cpp.md)) — the window outlives every graphics device, including across a full device-lost rebuild.

## `initialize`

**Contract** — Starts the global and the coarse wall clocks, chooses the application identity from the game generation being emulated, creates a hidden borderless resizable window at a placeholder size, installs the crash reporter's window hook and the window's hit-test callback, and registers the debug overlay into the frame sequence. Fails hard if the window cannot be created — there is no headless fallback; the dedicated server still creates a window and simply never shows a rendered frame in it.

```text
FUNCTION initialize()
  start global_timer          # pausable; the source of frame timestamps
  start wall_timer            # never pauses; used for real-time measurements

  window_flags = {borderless, hidden, resizable}
  renderer.add_required_window_flags(window_flags)   # the backend asks for e.g. a GL context

  (icon, title) = identity_for(game_generation)      # three shipped games, three titles and icons
  title = override_from_settings("window", "title") OR title
  publish title as the process's application title and to the host's audio/IME naming hints

  window = create_window(title, size = 640x480, window_flags)
  FAIL WITH "unable to create window" IF window is absent

  install hit_test_callback on window
  set minimum window size = 256x192
  register this device as the crash reporter's window owner
  extract icon resource from the executable image and attach it to the window

  IF NOT dedicated_server THEN register the debug overlay in the frame sequence at priority -5
```

**Notes** — The window is created **hidden** and at a throwaway size. The real size is not known yet: it depends on the saved display mode, which the resolution-selection pass in [`Device_mode.cpp`](Device_mode.cpp.md) resolves against the monitor's actual mode list, and the window is only shown once a graphics device has been created behind it. Showing a correctly-sized empty window first would produce a visible flash of desktop-coloured nothing.

**Notes** — It is created **borderless** regardless of the target mode, and borders are added back later if the final mode is a bordered window. This is cheaper than destroying and recreating the window, and a borderless-to-bordered transition does not lose the graphics device.

**Notes** — The host windowing library is asked for four behaviour changes: do not minimize a fullscreen window when it loses focus (the player alt-tabs constantly and a minimize costs a device reset), name the process in the audio mixer and the task list with the game's own title, show the host's input-method editor UI so non-Latin text entry works in the console, and do **not** auto-capture the mouse on button press because the engine owns capture itself and fights the library over it.

**Notes** — The debug overlay is registered at a negative priority so it runs *before* every game-side frame handler. It has to: it consumes input events and can claim them, and a handler that ran first would act on a key the overlay was going to eat.

**Notes** — The window title is overridable from the engine's own settings file. That is how a mod ships under its own name without a rebuild.

## `hit_test`

**Contract** — Answers, for a point in the window, what dragging there means: resize from one of eight edges and corners, move the whole window, or nothing special. Consulted by the host on every pointer press in a borderless window. Returns "nothing special" immediately unless the device currently allows dragging.

```text
FUNCTION hit_test(point) -> one of {none, move, resize_<edge or corner>}
  IF NOT window_draggable THEN RETURN none
  margin = 15 pixels
  left   = point.x <= client.x + margin
  top    = point.y <= client.y + margin
  right  = point.x >= client.w - margin
  bottom = point.y >= client.h - margin
  corners first, then edges, else move
```

**Notes** — A fifteen-pixel grab margin is the whole reason a borderless window is usable at all: with no frame there is no other way to resize it. The value is a feel decision — wide enough to hit with a mouse, narrow enough not to swallow clicks on UI near the edge.

**Notes** — A coordinate arriving near 65535 with a client width well below that is a negative coordinate that the host delivered unsigned; it is corrected by subtracting the wrap point. This is a defect in the windowing library, not a decision. A rebuild whose host delivers signed coordinates deletes the correction.

## `dump_statistics`

**Contract** — Writes the device's own four numbers into the on-screen statistics panel: engine time per frame, the smoothed frame rate, the smoothed *render* frame rate, and triangles per second in millions. Raises a performance alert when the frame rate falls below 30.

**Notes** — Two frame rates are reported because they can differ: one counts loop iterations and one counts presented frames, and they diverge when the loop runs simulation steps it does not render. Thirty is the alert threshold because it is half the sixty the engine targets — a frame rate below it is a failure, not a dip.

## `application_window_handle`

**Contract** — Reports the host's native window handle where one exists, and nothing elsewhere. Needed only by the Direct3D path, which binds a swap chain to a native handle; every other consumer works through the windowing abstraction.
