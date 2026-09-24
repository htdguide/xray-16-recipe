# src/Layers/xrRenderDX11/dx11HW.cpp

> Brings up the graphics device and its presentation chain, decides which capability level the machine can actually run, and owns the lost/reset lifecycle every other resource in the backend hangs off.

**Needs** — [`dx11HW.h`](dx11HW.h.md) · [`StateManager/dx11SamplerStateCache.h`](StateManager/dx11SamplerStateCache.h.md) · [`dx11TextureUtils.h`](dx11TextureUtils.h.md) · [`xrRender/HWCaps.h`](../xrRender/HWCaps.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`dx11HW.h`](dx11HW.h.md)
**Tier floor** — T1: it resolves driver entry points out of dynamically loaded libraries by name, hands the presentation layer an OS window handle, and must release device objects in a defined order at a defined time.

## Purpose

This is the one place in the backend that talks to the *driver* rather than to the graphics API's drawing surface. It loads the driver libraries, picks an adapter, negotiates the highest capability level the adapter will grant, creates the submission contexts (one immediate plus a fixed pool of deferred ones), attaches a presentation chain to the window the platform layer opened, and chooses the back-buffer and depth formats by asking the driver which candidates it supports rather than assuming.

It is a separate file because everything else in the backend assumes a live device exists. Its second job — the one that is easy to miss and expensive to retrofit — is the **device-state machine**: presentation reports occlusion, reset-required and device-removed, and the frame loop consults this module once per frame to decide whether to render, rebuild or die.

Exactly one instance of this is the process-wide device. The class is nonetheless written so a second, non-global instance can be constructed (the editor host does this to probe hardware without taking over the application); every action with a global side effect — registering for activate/deactivate notification, creating deferred contexts, driving fullscreen transitions, aborting the process on failure — is gated on "am I the global one".

## State

```text
RECORD GraphicsDevice
  valid            : bool          # false means "this device did not come up"; callers must check
                                   # before touching anything else, and a failed non-global
                                   # instance must return rather than abort the process
  adapter          : handle        # display adapter 0; no multi-adapter selection exists
  device           : handle        # the resource-creating object
  contexts         : list<handle>  # fixed size: N parallel deferred + 1 immediate
  immediate_index  : int           # == count of parallel contexts, i.e. the last slot
  swap_chain       : handle
  chain_desc       : record        # width, height, back-buffer format, windowed flag, buffer count
  back_buffer_count: int           # 1 with a discard-style present; a flip-style present needs >= 2
  current_back_buffer : int        # invariant: always < back_buffer_count
  feature_level    : enum          # the capability tier the driver granted
  caps             : HardwareCaps  # vendor/device id, chosen depth format, chosen target format
  compute_supported: bool
  do_present_test  : bool          # set when a present reported occlusion or removal; makes the
                                   # next state query run a no-op present to re-probe
  compile_entry    : function      # the shader compiler entry point, resolved at run time
```

Invariants: `caps.depth_format` and `caps.target_format` are filled before any render target is created, because the render-target set is described in terms of them. `contexts[immediate_index]` is the only context that may submit directly; the others record and are replayed.

## `CreateDevice`

**Contract** — takes the platform layer's window, brings the whole device up, and leaves `valid` true on success. Blocks. On failure in the global instance it reports and terminates the process: there is no recovery path at startup, and pretending otherwise produces a black window. On failure in a non-global instance it returns quietly with `valid` false — this is how the renderer-selection step (chapter 20's entry module) asks "would this backend work here?" without killing the application.

```text
FUNCTION create_device(window) -> ()
  load_driver_libraries()                 # by name, at run time, not linked
  IF libraries missing THEN valid = false; RETURN
  adapter = first adapter of the factory  # adapter 0, always
  record vendor id, device id, adapter name, dedicated video memory

  # Capability negotiation: ask for the newest tier first and walk down.
  # Two lists, not one, because some drivers fail the whole call when asked
  # for a tier they do not know, rather than granting a lower one.
  IF forced_legacy THEN levels = [legacy tiers]
  ELSE
    levels = [newest .. oldest]
    IF creation fails THEN levels = [tiers the previous generation of drivers knows]
  device, granted_level, immediate_context = create(adapter, levels)
  IF failed THEN
    valid = false
    IF global THEN FAIL WITH "graphics hardware could not be initialized"
    RETURN

  compute_supported = granted_level >= tier_11
                      OR driver reports the compute-on-older-tier option
  query optional shader features (double precision, packed-difference instruction)

  # Shader compiler entry point, resolved rather than linked (see Notes)
  compile_entry = system compiler
  IF granted_level < tier_11 AND running the middle game's data THEN
    compile_entry = entry of the period-correct compiler library

  IF global THEN
    FOR EACH i IN 0 .. parallel_context_count - 1
      contexts[i] = create_deferred_context()

  handle = native handle of window          # from the windowing seam
  IF NOT create_modern_swap_chain(handle) THEN
    IF NOT create_legacy_swap_chain(handle) THEN valid = false

  caps.depth_format = first candidate the driver reports usable as a depth-stencil target
  IF none THEN valid = false; IF global THEN FAIL WITH "no usable depth format"
```

**Notes** — The driver libraries are loaded by name and their entry points looked up, never link-time bound. Two reasons, both load-bearing: the executable must start on a machine where this backend is unavailable (so that the entry module can fall back to the other backend instead of failing to load), and third-party translation layers that re-implement the API as a library are supported by simply being that library. The matching teardown explicitly unloads them, because a translation layer only releases its own resources when its library is unloaded.

The shader-compiler entry point is a *run-time resolved function*, not a compile-time dependency, for the same reason plus one more: one of the three shipped games ships shader sources that only the period-correct compiler accepts, and that case is selected by the data set being run, not by the hardware.

## `CreateSwapChain` / `CreateSwapChain2`

**Contract** — attach a presentation chain to the window. Two implementations exist, newer first; the newer one is skipped entirely on a command-line switch, and either may fail without that being fatal so long as one succeeds. Both select the back-buffer format by probing candidates for display support rather than assuming one, and both record the chosen format into the capability record.

The newer path additionally exposes a *presentation-finished* waitable signal to the frame loop, which is how the loop avoids queueing frames ahead of the display; the older path has no such signal and the loop falls back to timing.

**Notes** — Both paths currently request a discard-style present with a single back buffer even though the newer path could use a flip model. The source says flip presentation produced tearing artifacts and leaves the flip constant commented out; the format candidate lists are likewise one entry long with wider-precision candidates commented out, one of them because the screenshot path cannot read that format back. A rebuild should treat the candidate list as a genuine list — the probing machinery is already there — and re-test the flip model rather than inheriting the workaround.

## `Reset`

**Contract** — re-establish the chain after the window's size or fullscreen state changed. Sets the windowed flag, drives the fullscreen transition, resizes the target and then the buffers. Everything that referenced the old back buffer is invalid afterwards, which is why the frame loop tears down and rebuilds the render-target set around a call to this.

## `GetDeviceState`

**Contract** — answers `Normal`, `Lost` or `NeedReset`, and is consulted once per frame before any rendering work. Cheap in the common case: it only probes when a previous present flagged trouble.

```text
FUNCTION device_state() -> {Normal, Lost, NeedReset}
  IF NOT do_present_test THEN RETURN Normal
  result = present(no flip, test only)
  IF result is ok            THEN do_present_test = false; RETURN Normal
  IF result is occluded      THEN RETURN Lost        # window hidden: skip the frame entirely
  IF result is reset_needed  THEN RETURN NeedReset   # rebuild every device resource
  IF result is device_gone   THEN FAIL WITH "driver replaced or adapter removed; restart required"
```

**Notes** — The three outcomes are genuinely different and a filling of this seam must distinguish them. *Occluded* means do no work and do not rebuild — a minimized or fully covered window, which happens constantly. *Reset needed* means every device-owned resource must be recreated from its source data, which is why the whole backend keeps enough information to rebuild each resource rather than only the resource. *Removed* is unrecoverable within the process.

## `Present`

**Contract** — flips the chain and advances the back-buffer index modulo the buffer count. Vertical sync is requested only in exclusive fullscreen; in windowed mode it is deliberately not requested, because the composited windowed path produced tearing with it on. Occlusion or removal reported here arms the state probe rather than acting immediately, so the failure is handled at one place at the top of the next frame.

## `OnAppActivate` / `OnAppDeactivate`

**Contract** — the application's focus notifications. Losing focus while in exclusive fullscreen drops to windowed and minimizes; regaining focus restores and re-enters fullscreen. Only the global instance subscribes. This exists because an exclusive-fullscreen presentation chain owns the display mode, and a background process must not.

## `DestroyDevice`

**Contract** — releases in a fixed order: the cached state-object arrays first (they hold device objects and would otherwise outlive the device), then the presentation chain — after forcing windowed mode, because an exclusive-fullscreen chain cannot be released — then the contexts, then the device, then the driver libraries. Ordering here is the contract; a rebuild in a language with non-deterministic finalization must still force this order explicitly.

## `CheckFormatSupport` / `SelectFormat`

**Contract** — ask the driver whether a pixel format supports a named use (display, depth-stencil, blending, filtering, render target), and pick the first candidate from an ordered list that does. Every format decision in the backend is expressed this way: an ordered preference list plus a capability question, never a hard-coded format. This is the mechanism a rebuild needs most, because a different graphics API will have a different set of universally available formats.

## `GetSurfaceSize`, `UsingFlipPresentationModel`, `BeginScene`, `EndScene`, `SetPrimaryAttributes`

**Contract** — accessors and stubs. The surface size is read back from the chain rather than from the requested size, because the driver may have adjusted it. The scene bracket is empty here (it exists because the older backend generation required it) and the window-attribute hook contributes nothing on this backend — the other backend uses it to ask the windowing layer for a graphics context, and this one attaches to the raw window handle instead.
