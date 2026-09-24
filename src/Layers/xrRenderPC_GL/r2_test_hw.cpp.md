# src/Layers/xrRenderPC_GL/r2_test_hw.cpp

> The device probe: decides whether this backend can run at all, by actually building a context on a throwaway window.

**Needs** — [`../xrRenderGL/glHW.h`](../xrRenderGL/glHW.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xrRender_GL.cpp`](xrRender_GL.cpp.md)
**Tier floor** — T1: it creates and destroys a real device context as a side effect.

## Purpose

The renderer-selection path needs to know, before offering this backend to the player, whether the machine can run it. The decision made here is that **no amount of capability querying substitutes for trying**: rather than inspect a version string or an extension list, the probe creates a hidden one-pixel window with this backend's context requirements, creates a device on it, and checks that the device came up. Then it tears both down.

That is the right answer for this class of question and a rebuild should copy it. A driver that advertises the required version and then fails to give you a core-profile context of that version is common enough that the string-inspection approach mis-reports; a driver that gives you a working context is, by construction, one you can run on.

## State

```text
# All state is scoped to one probe and released before it returns.
RECORD Probe
  window : handle to a hidden 1×1 window
  device : a NON-GLOBAL device instance      # see glHW.cpp: the global/non-global
                                             # distinction exists for exactly this
```

## `test_device`

**Contract** — returns whether this backend can create a working device on this machine. Blocks — context creation and entry-point resolution are the expensive part. Leaves nothing behind: the window and the context are destroyed before the answer is returned. Logs, but does not fault, when the window itself cannot be created.

```text
FUNCTION test_device() -> bool
  flags := {}
  device.set_primary_attributes(flags)          # the same requirements the
                                                # real window will be given
  window := windowing.create_window("probe", 1×1, hidden, flags)
  IF window IS none THEN
      log "cannot create helper window"; RETURN false

  device.create_device(window)                  # non-global instance

  success := window EXISTS
             AND device.context EXISTS
             AND device.framebuffer EXISTS

  device.destroy_device()
  windowing.destroy_window(window)
  RETURN success
```

**Invariants** — the three-part success test is the meaningful part. A context alone is not enough: the probe also requires that the offscreen framebuffer was created, which only happens if entry-point resolution succeeded and the device offers framebuffer objects. Those are precisely the two failures a version string will not reveal.

**Invariants** — the probing device instance must not be the global one. The global instance subscribes to application focus events and owns the vertical-sync setting; a probe that used it would unsubscribe a live device at teardown. The distinction is made in [`glHW.cpp`](../xrRenderGL/glHW.cpp.md) by comparing against the global instance's identity, which a rebuild should express as a constructor flag instead.

**Notes** — the probe runs *before* the engine's real window exists, so it must not assume any windowing state. It also runs on every startup, adding the cost of one context creation to launch. A rebuild could cache the answer against the adapter identity, in the same spirit as the shader binary cache.
