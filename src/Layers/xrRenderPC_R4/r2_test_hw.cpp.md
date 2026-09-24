# src/Layers/xrRenderPC_R4/r2_test_hw.cpp

> The device probe: builds a real device on a throwaway window and reports not just whether this backend runs, but *which tier of it* runs.

**Needs** — [`../xrRenderDX11/dx11HW.h`](../xrRenderDX11/dx11HW.h.md) · [`xrRender_R4.cpp`](xrRender_R4.cpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xrRender_R4.cpp`](xrRender_R4.cpp.md)
**Tier floor** — T1: it creates and destroys a real device as a side effect, and reads a capability tier off it.

## Purpose

Before the player is offered this backend, somebody has to answer "will it run here". The decision made here is the same one the sibling backend makes — **no capability query substitutes for trying** — with one addition that is specific to this filling and is the whole reason the file is worth reading: the probe returns a **three-valued** answer, not a yes/no.

The three values are *not supported*, *supported at the older capability tier*, and *supported at the newer capability tier*. That is because this one module ships several renderer names at different quality levels, and which of them may be offered depends on the tier the driver grants. Collapsing the probe to a boolean would make the module offer its highest quality level on hardware that cannot run it.

## State

```text
# Everything is scoped to one probe and released before it returns.
RECORD Probe
  window : handle to a hidden 1×1 window
  device : a NON-GLOBAL device instance    # see dx11HW.cpp: the global/non-global
                                           # split exists for exactly this caller
```

## `test_device`

**Contract** — returns the capability tier this machine grants, as one of three outcomes. Blocks: it creates a device, which is the expensive part of startup. Leaves nothing behind — the device and the window are destroyed before the answer is returned. Logs, but does not fault, when the window cannot be created; a machine with no display is a legitimate "not supported", not a crash.

```text
ENUM ProbeResult = unsupported | tier_older | tier_newer

FUNCTION test_device() -> ProbeResult
  window := windowing.create_window("probe", 1x1, hidden)
  IF window IS none THEN
      log "cannot create helper window"; RETURN unsupported

  device.create_device(window)          # non-global: failure returns, does not abort

  IF window EXISTS AND device.valid THEN
      tier := device.granted_capability_tier
  ELSE
      tier := none

  device.destroy_device()
  windowing.destroy_window(window)

  IF tier >= the newer tier THEN RETURN tier_newer
  IF tier >= the older tier THEN RETURN tier_older
  RETURN unsupported
```

**Invariants** — the probing device must be the **non-global** instance. The global one aborts the process when creation fails, subscribes to application focus notifications and owns the presentation settings; a probe that used it would either kill the application on an unsupported machine or unsubscribe a live device at teardown. [`dx11HW.cpp`](../xrRenderDX11/dx11HW.cpp.md) gates every globally-visible action on that distinction.

**Invariants** — the tier is read from what the driver **granted**, not from what was asked for. Device creation negotiates downward through a list of tiers, so a machine that cannot do the newer one still produces a valid device at the older one, and the probe must report that rather than treating it as failure.

**Notes** — the three-valued result is carried in the source as a truth value *plus one*, which is flagged there as a hack awaiting a proper enumeration. A rebuild should name the three outcomes. What must survive is that there are three and the caller branches on all of them — see [`xrRender_R4.cpp`](xrRender_R4.cpp.md), where the tier decides which renderer names appear in the menu.

**Notes** — the probe runs at every startup and costs one full device creation. It runs before the engine's real window exists, so it may assume no windowing state. A rebuild could cache the answer against the adapter identity, in the same spirit as the compiled-shader cache in [`r4_shaders.cpp`](r4_shaders.cpp.md).
