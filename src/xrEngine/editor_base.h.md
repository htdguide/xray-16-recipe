# src/xrEngine/editor_base.h

> Declares the debug overlay shell and the interface a tool must satisfy to live in it.

**Needs** — [`editor_base.cpp`](editor_base.cpp.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`pure.h`](pure.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`Device_imgui.cpp`](Device_imgui.cpp.md) · [`Environment.h`](Environment.h.md) · [`XR_IOConsole.cpp`](XR_IOConsole.cpp.md) · [`device.cpp`](device.cpp.md) · [`device.h`](device.h.md) · [`editor_base.cpp`](editor_base.cpp.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) · [`object_factory.h`](../xrServerEntities/object_factory.h.md)
**Tier floor** — T3: a declaration and one interface.

## Purpose

Declares the shell implemented in [`editor_base.cpp`](editor_base.cpp.md) and
[`editor_base_input.cpp`](editor_base_input.cpp.md), and — substantively — the **tool
interface**, which is a contract a rebuild's tools must satisfy.

## `ide_tool` — what a tool must provide

**Contract** — a tool registers itself with the overlay on construction and unregisters on
destruction; it need do nothing else to appear in the Tools menu. It owns its own open
state, which the menu's checkbox writes directly.

```text
INTERFACE Tool
  on_tool_frame()                   # REQUIRED. called every frame in full and light states
  tool_name() -> text               # REQUIRED. the menu entry and the settings section name

  open_state -> bool (mutable)      # the menu's checkbox writes this
  is_active() -> bool               # default: open. override to mean "doing work"

  default_window_flags() -> flags   # the shell's; a tool must use these to honour light mode

  # settings persistence, all optional
  reset_settings()                  # forget everything
  apply_setting(line)               # one line read back from the settings file
  apply_settings()                  # all lines delivered
  save_settings(out buffer)         # append this tool's lines
  estimate_settings_size() -> int   # so the writer can reserve once
```

**Notes** — `on_tool_frame` is called in the **light** state too, so a tool must decide for
itself whether it draws when it is not open. Most test their open state and return.

The settings hooks form a section-per-tool file managed by the toolkit, and the estimate
exists only so the writer reserves its buffer once; a rebuild whose buffer grows can return
zero. The comment in the source notes that the apply-all hook is near-useless because
settings are applied as each line arrives — which is the more robust design and the one a
rebuild should keep.

`is_active` defaulting to the open state, and being overridable, lets a tool say it is doing
work while closed — a profiler still sampling, say — which is what keeps the state cycle
from hiding it.

## `ide` — the shell

Declares the surface contracted in [`editor_base.cpp`](editor_base.cpp.md) (the three-state
model, the tool list, the menu bar) and in
[`editor_base_input.cpp`](editor_base_input.cpp.md) (the backend half: event translation,
cursor, text-input mode, viewport tracking).

Its exported units: the three visibility states and the transitions between them
(`SetState`, `SwitchToNextState`, `GetState`, `IsActiveState`, `is_shown`); `InitBackend`;
`ProcessEvent`; `UpdateTextInput`; the update-phase and focus-sequence hooks; and the full
input-receiver surface — mouse, keyboard, text and controller — which it implements by
translating each into the toolkit's event queue.
