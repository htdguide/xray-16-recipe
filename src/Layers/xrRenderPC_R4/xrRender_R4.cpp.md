# src/Layers/xrRenderPC_R4/xrRender_R4.cpp

> The module entry point: announces the renderer names this backend answers to, refuses to offer them when the device or the data will not support them, and on selection caps the device tier and publishes this backend into the engine's global environment.

**Needs** — [`r2_test_hw.cpp`](r2_test_hw.cpp.md) · [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) · [`../xrRenderDX11/dx11HW.h`](../xrRenderDX11/dx11HW.h.md) · [`../xrRender/dxRenderFactory.h`](../xrRender/dxRenderFactory.h.md) · [`../xrRender/dxUIRender.h`](../xrRender/dxUIRender.h.md) · [`../xrRender/dxDebugRender.h`](../xrRender/dxDebugRender.h.md) · [`../xrRender/D3DUtils.h`](../xrRender/D3DUtils.h.md) · [`../xrRender/xrRender_console.h`](../xrRender/xrRender_console.h.md) · [`../../Include/xrRender/xrRender.h`](../../Include/xrRender/xrRender.h.md) · [`../xrAPI/README.md`](../xrAPI/README.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`r2_test_hw.cpp`](r2_test_hw.cpp.md)
**Tier floor** — T2: a list, four callbacks and a handful of assignments; it lives in a T1 module only because of what it points at.

## Purpose

The [graphics-device seam](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) says a renderer is *selected at startup by name, and a failed device creation falls back to the next candidate*. This file is this filling's side of that contract. It is short, and everything in it is load-bearing, because it is the only place where the seam's own mechanism is visible from inside a filling.

The idea a rebuilder must take away is that **one module offers several names**. The names are not separate backends; they are quality presets over the *same* code, and the difference between them is two switches plus one lighting decision. That is why the seam interface is "return a list of (name, rank)" rather than "return your name": the number of selectable renderers the player sees is not the number of backends the engine has.

## The names, and what they mean

Frozen user-facing vocabulary. They appear in the shipped configuration files, in the settings the player writes back, and in the quality menu, so a rebuild may not invent its own without also honouring these.

| Name | Rank | Device tier | Advanced post | Sun |
|---|---|---|---|---|
| `renderer_r2a` | 1 | legacy only | off | **baked into lightmaps** |
| `renderer_r2` | 2 | legacy only | off | dynamic |
| `renderer_r2.5` | 3 | legacy only | on | dynamic |
| `renderer_r3` | 4 | legacy only | on | dynamic |
| `renderer_r4` | 5 | newest | on | dynamic |

The rank orders the quality presets in the menu and is what the engine stores; the name is what it matches on. `renderer_r2a` is present in the source but **not offered**: the shipped shader sources of the original game fail to compile with static sun enabled, and the mode exists only for data sets that supply their own.

Ranks 2 through 4 are indistinguishable at the device level — all three cap the device to the legacy tier — and differ only in whether the advanced post-processing group is allowed. They are three separate menu entries because the shipped game's quality presets have three entries there, not because the engine has three code paths.

## State

```text
RECORD RendererModule
  modes : list<(name, rank)>     # empty until first asked; built once
# Invariant: built exactly once. A second query must return what the first
# produced rather than appending again; teardown clears it.
```

## `supported_modes`

**Contract** — returns the renderer names this backend offers on this machine, building the list on first call. Blocks on the first call, because it runs the device probe. Returns an empty list when the machine cannot run this backend at all, which is how the engine learns to skip this module entirely.

```text
FUNCTION supported_modes() -> list<(name, rank)>
  IF modes is non-empty THEN RETURN modes        # never offer twice

  tier := test_device()                          # three-valued; see r2_test_hw.cpp
  IF tier IS unsupported THEN RETURN modes       # empty

  offer ("renderer_r2",   2)
  offer ("renderer_r2.5", 3)
  IF tier IS tier_older THEN
      offer ("renderer_r3", 4)
  ELSE                                            # tier_newer
      offer ("renderer_r3", 4)                    # first
      offer ("renderer_r4", 5)                    # then
  RETURN modes
```

**Invariants** — **insertion order is the presentation order.** The engine concatenates every module's list in the order given and the menu shows it in that order; the higher-quality name must come second or the quality ladder reads backwards. The source makes the point by writing the two-name case out longhand rather than letting the lower case fall through into it, with a comment forbidding the tidier spelling.

**Invariants** — the three lower names are gated on the probe succeeding *at all*, not on any tier. A machine that cannot create a device of either tier is offered nothing from this module, not a degraded preset — there is no software path here.

## `check_game_requirements`

**Contract** — answers whether the installed game data can feed this backend. Checks exactly one thing: that this backend's shader source directory exists under the game's shader root. Returns false with a log line if not.

**Invariants** — this backend reads the **shipped** shader tree, the one the retail game already contains for its own highest renderer. That is the deepest difference between this filling and the OpenGL one, which needs a parallel tree added by the engine's own data package (see [`../xrRenderPC_GL/rgl_shaders.cpp`](../xrRenderPC_GL/rgl_shaders.cpp.md)). A rebuilder filling this seam with a third API inherits the OpenGL backend's problem, not this one's: the shipped sources are in the dialect this backend's compiler already speaks, and nothing else is.

**Notes** — the check is a *directory existence* test, not a validation. A truncated or wrong-version shader tree passes here and fails later, one material at a time, at level load.

## `setup_environment(mode)`

**Contract** — called once, on the module whose name was chosen. Sets the switches the mode implies, then publishes this backend's implementations into the engine's global environment. Runs **before** the device is created.

```text
FUNCTION setup_environment(mode) -> ()
  static_sun := false
  SELECT mode
    "renderer_r2a"   -> static_sun := true      # then fall into the next case
                        cap_device_to_legacy_tier := true
                        advanced_post := false
    "renderer_r2"    -> cap_device_to_legacy_tier := true
                        advanced_post := false
    "renderer_r2.5",
    "renderer_r3"    -> cap_device_to_legacy_tier := true
                        advanced_post := true
    "renderer_r4"    -> advanced_post := true

  environment.render          := this backend's renderer
  environment.render_factory  := this backend's object factory
  environment.draw_utilities  := this backend's drawing utilities
  environment.ui_render       := this backend's UI renderer
  IF debug build THEN
      environment.debug_render := this backend's debug renderer, and register it
  install_renderer_console_variables()
```

**Invariants** — **the ordering is the load-bearing part.** Mode selection happens before device creation, and `cap_device_to_legacy_tier` is read by device creation to decide which tiers it will even ask the driver for. A player who picks a lower preset therefore gets a genuinely lower-tier device, not a high-tier device driven conservatively. A rebuild that creates the device first and configures afterwards loses this, and with it the ability to reproduce the older presets' behaviour on modern hardware — which is the point of shipping them.

**Invariants** — the mode string is dispatched through a compile-time hash of the name. Incidental in itself; what survives is that the *name* is the key, so the set of names is the interface and adding a preset means adding a name.

**Notes** — the five published references are the real content of this section: they are the complete list of what a graphics backend must supply beyond the renderer proper — the renderer, a factory for render-side objects the game layer creates, the immediate-mode drawing helpers, the user-interface renderer, and (in a debug build) the debug-geometry renderer. This is the service-locator cycle-breaker described in [`../xrAPI/README.md`](../xrAPI/README.md); a rebuild should make them explicit dependencies, but must supply all five.

## `clear_environment`

**Contract** — teardown. Clears the mode list, then clears every published reference **only if the renderer reference still names this backend**.

**Invariants** — that guard is not defensive clutter. Renderer modules are enumerated in sequence at startup, every module is asked for its modes, and the modules that were *not* chosen are torn down afterwards. Without the guard, a probed-but-unchosen module would unpublish the chosen one and the engine would start with no renderer. A rebuild that keeps a service locator must keep an equivalent check; one that passes the chosen renderer explicitly does not need it.

## `renderer_module`

**Contract** — the module's single exported symbol: returns the one module object, which has static storage and no lifetime of its own. This is the whole of the module interface — everything above is reached through the four callbacks on the returned object.

**Notes** — the symbol is exported only on the platform that has this API at all; on every other platform the module is not built and the engine's candidate list is one shorter. The candidate list is a fixed-size array in the engine, so "how many fillings exist" is a build-time fact even though "which one runs" is not.
