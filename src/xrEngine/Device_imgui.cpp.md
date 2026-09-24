# src/xrEngine/Device_imgui.cpp

> Stands up the debug overlay toolkit against the engine's allocator, window and clipboard, and tears it down again.

**Needs** — [`device.h`](device.h.md) · [`editor_base.h`](editor_base.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`ImGuiRender.h`](../Include/xrRender/ImGuiRender.h.md)
**Tier floor** — T1: it installs allocator hooks and creates native child windows.

## Purpose

The debug overlay toolkit is a guest in this process. It does not know how to allocate, where to put its settings, how the clipboard works, or how to make a window — the engine supplies all four. This file is that supply, and it is entirely the *platform* half of the integration; the drawing half is the backend's and the widget content is the debug tooling's.

The whole file is optional. A shipping rebuild that drops the overlay seam drops this file with it.

## `initialize_overlay`

**Contract** — Idempotent; returns if a context already exists. Routes the toolkit's allocation through the engine allocator, creates its context, enables keyboard navigation, gamepad navigation and docking, redirects its two persistence files into the engine's virtual filesystem, installs clipboard and text-composition callbacks, hands the toolkit a set of window-management callbacks when detached overlay windows are enabled, and asks the debug tooling to bring up the drawing backend. Enables detached windows only if the backend reports it can support them.

```text
FUNCTION initialize_overlay()
  IF context exists THEN RETURN
  install allocator hooks -> engine allocator
  context = create_overlay_context()
  enable keyboard navigation, gamepad navigation, docking
  enable "move the pointer when navigation moves focus"

  settings_file = resolve("$app_data_root$", toolkit's default settings name)
  log_file      = resolve("$logs$",          toolkit's default log name)

  install clipboard get/set -> host clipboard
  install text-composition placement -> host IME rectangle, positioned one line below the caret

  IF detached overlay windows are compiled in THEN
    install window create/destroy/show/move/resize/focus/minimize/title/opacity callbacks

  debug_tooling.initialize_backend()
  IF the backend reports viewport support THEN enable detached windows
```

**Invariants** — The allocator hooks are installed **before** the context is created, since creating it allocates. Getting this order wrong means a block allocated by one allocator and freed by the other.

**Notes** — Routing the toolkit through the engine allocator is not a nicety. The engine's allocator is a seam with three possible fillings, and one of them tracks every block; a guest library allocating around it produces leak reports that are wrong and heap statistics that do not add up.

**Notes** — The settings and log paths go through the virtual filesystem's logical roots rather than the working directory, so the overlay's remembered window layout lands next to the player's other settings and survives a reinstall. Separator normalization is applied because the toolkit writes the path out verbatim.

**Notes** — The clipboard read callback frees the previously returned string before fetching a new one. The toolkit's contract is that the returned text stays valid until the next call, so one slot of ownership is held on the engine's side — a rebuild in a language with owned strings returns a value and deletes this entirely.

**Notes** — The text-composition rectangle is placed one line *below* the caret, not at it, so the host's candidate window does not cover the character being typed.

**Notes** — Assertion-based error recovery, error tooltips and duplicate-identifier highlighting are enabled only in a debug build. In a release build an overlay bug must not take the game down.

**Notes** — Detached overlay windows are a compile-time option and the platform half is substantial: create a window with the backend's required flags, hidden, decorated or not according to the toolkit's request, optionally skipping the task bar and optionally always on top; parent it modally when the toolkit names a parent. Two of these need host-specific escapes — hiding a window from the task bar, and showing a window *without* activating it, which the windowing library always does. Those escapes are bugs worked around, not decisions; a rebuild whose windowing layer can express both drops them.

## `destroy_overlay`

**Contract** — Idempotent. Releases the drawing backend through the render factory that made it, destroys any detached windows, releases the two redirected path strings and the clipboard's held string, and destroys the context. Order matters: the backend holds graphics resources bound to the context and must go first.
