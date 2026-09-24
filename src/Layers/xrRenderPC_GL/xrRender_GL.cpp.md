# src/Layers/xrRenderPC_GL/xrRender_GL.cpp

> The module entry point: announces which renderer names this backend answers to, refuses to offer them when the device or the data will not support it, and wires the global environment when one is chosen.

**Needs** — [`r2_test_hw.cpp`](r2_test_hw.cpp.md) · [`../xrRender_R2/r2.h`](../xrRender_R2/r2.h.md) · [`../../Include/xrRender/xrRender.h`](../../Include/xrRender/xrRender.h.md) · [`../../Layers/xrAPI/README.md`](../xrAPI/README.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is a registration and a handful of assignments; it sits in a T1 module only because of what it points at.

## Purpose

The seam says a renderer is "selected at startup by name, and a failed device creation falls back to the next candidate". This file is this backend's side of that contract, and it is short because the contract is small: offer a list of names, answer two yes/no questions, and on selection publish a set of pointers into the engine's global environment.

The interesting content is the *conditions*, because they are where a rebuilder learns what has to be true before this backend can run at all, and because the offered name list differs by platform in a way that looks arbitrary and is not.

## State

```text
RECORD RendererModule
  modes : list<(name, numeric id)>    # empty until first asked
# Invariant: the list is built once and never duplicated. A second query
# returns what the first produced; clearing it is part of teardown.
```

## `supported_modes`

**Contract** — returns the renderer names this backend offers, building the list on first call. Offers nothing if a list already exists, or if the device probe fails. Blocks on the first call, because the probe creates and destroys a real context.

```text
FUNCTION supported_modes() -> list<(name, id)>
  IF modes is non-empty THEN RETURN modes          # never offer twice
  IF NOT probe_device() THEN RETURN modes          # see r2_test_hw.cpp

  IF on the platform that also ships the Direct3D backend THEN
      offer ("renderer_rgl", 6)
  ELSE
      offer ("renderer_r3", 4)
  RETURN modes
```

**Invariants** — the name-and-number pairs are a frozen user-facing vocabulary: they appear in the shipped configuration files and in the settings the player writes back, and the numbers order the quality presets. This backend claims a *distinct* name on the platform where the Direct3D backend also exists, so the player can choose between them; on every other platform it claims the *existing* mid-tier name instead, because that is the name the shipped configuration and the menus already reference and nothing else is there to claim it. Two lower names and one higher one are present in the source, commented out — the backend can technically run them and they are not offered because they are untested.

**Notes** — the consequence for a rebuild is that "which renderer am I" is answered by a name that the data also uses, so a rebuild may not invent its own names without also handling the shipped ones.

## `check_game_requirements`

**Contract** — answers whether the installed game data can feed this backend. Checks exactly one thing: that this backend's own shader directory exists. Returns false with a log line if not.

**Invariants** — this is where the "ships replacements rather than translating" decision from [`rgl_shaders.cpp`](rgl_shaders.cpp.md) becomes a startup requirement. The shipped game does not contain this directory; it is added by the engine's own data package. Without it the backend must not be offered, because every material would fail to compile.

## `setup_environment(mode)`

**Contract** — called once when this backend's mode is chosen. Sets the two renderer-level switches the mode implies, then publishes this backend's implementations into the engine's global environment — the renderer itself, the render-object factory, the drawing utilities, the user-interface renderer, and (in a debug build) the debug renderer, which also registers its console commands. Finally installs the renderer's own console variables.

```text
FUNCTION setup_environment(mode) -> ()
  static_sun        := false           # this backend always lights the sun dynamically
  advanced_post     := true            # default; one mode turns it off

  SELECT mode
    the plain second-generation name -> advanced_post := false
    every other name this backend claims -> advanced_post := true

  environment.render          := this backend's renderer
  environment.render_factory  := this backend's object factory
  environment.draw_utilities  := this backend's drawing utilities
  environment.ui_render       := this backend's UI renderer
  IF debug build THEN
      environment.debug_render := this backend's debug renderer, and register it
  install_renderer_console_variables()
```

**Notes** — this is the service-locator cycle-breaker described in [§7](../../../SYSTEM-REQUIREMENTS.md#7-build-order): the rest of the engine reaches the renderer through one mutable global filled here. A rebuild should make these explicit dependencies; the recipe notes at each use site what is being reached for. The *set* of pointers is the real content — it is the full list of what a graphics backend must supply beyond the renderer proper.

## `clear_environment`

**Contract** — teardown. Clears the mode list, and clears every published pointer **only if the renderer pointer still names this backend** — a guard that matters because renderer modules are enumerated in sequence at startup and a module that was probed but not chosen must not unpublish the one that was.

## `renderer_module`

**Contract** — the module's single exported symbol: returns the one module object. This is what the executable looks up after loading the backend's dynamic library, and it is the whole of the module interface.
