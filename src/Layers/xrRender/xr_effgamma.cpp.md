# src/Layers/xrRender/xr_effgamma.cpp

> Turns the three player-facing picture controls into a 256-entry colour ramp and pushes it at the display, not at a shader.

**Needs** — [`xr_effgamma.h`](xr_effgamma.h.md) · [`xrEngine/Device.h`](../../xrEngine/Device.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`xr_effgamma.h`](xr_effgamma.h.md)
**Tier floor** — T1: it hands a raw array of 16-bit ramp entries to the display output, whose length and element width the platform fixes.

## Purpose

Gamma, brightness and contrast are three sliders in the options screen. This file is the
whole of what they do: evaluate one curve per ramp entry and install the result on the
**display output**, downstream of everything the renderer draws.

That placement is the decision worth carrying. The ramp is not a post-processing pass and
does not appear in any shader: it is the output lookup table the display applies to the
already-presented frame, which means it costs nothing per frame, applies identically to
every renderer backend, and — the reason it was done this way — applies to the loading
screens and the menu as well as to the game. The cost is that it is a *display* setting:
it is lost whenever the device or the display mode changes, and must be re-asserted.

## State

```text
RECORD GammaControl
  gamma      : real    # exponent control; the curve is x^(1/gamma)
  brightness : real    # additive, in units where 1 is neutral
  contrast   : real    # multiplicative about mid-grey; 1 is neutral
  balance    : colour  # per-channel multiplier, default (1,1,1)
```

**Invariants**

- **All three scalars are neutral at 1.0**, and the console clamps each to `[0.5, 1.5]`
  (the clamp lives with the console variables in [`xrEngine/xr_ioc_cmd.cpp`](../../xrEngine/xr_ioc_cmd.cpp.md),
  not here). At `(1, 1, 1)` the generated ramp is the identity curve exactly — the
  constant offsets in the formula below are arranged so that they cancel, which is what
  makes "all defaults" mean "do not touch the picture".
- The record is the *requested* setting, never a readback. Nothing reads the installed
  ramp; if the display refuses it, the record and the screen disagree silently.
- `balance` is applied but never set: it is fixed at neutral by construction and no caller
  changes it. See Notes.

## `update()`

**Contract** — generates the ramp from the current settings and installs it. Takes and
returns nothing, reports nothing: a display that will not accept a ramp is not an error
condition, it is a display without the feature, and the picture stays uncorrected.

Called on two occasions, and both are load-bearing: when the player changes one of the
three settings, and again when a **level finishes precaching**, because bringing the
device up for a new level drops the ramp the display was holding.

```text
FUNCTION update()
  # Preferred: the display output behind the swap chain owns a ramp of its own,
  # with a device-reported control-point count and value range.
  IF the graphics device exposes its containing display output
    caps = that output's gamma capabilities
    IF caps were obtained                       # only true while fullscreen
      ramp = generate_ramp_in_output_space(caps)
      IF the output accepts ramp   RETURN

  # Fallback: the window system's own per-window ramp.
  ramp = generate_ramp(256 entries)
  install ramp on the window
```

**Notes** — The two paths exist because the display-output ramp is only obtainable while
the window owns the display exclusively; in a window the request fails and the
window-system ramp is the only route. A rebuild needs both only if it wants the fullscreen
path's higher precision — the device path uses a device-chosen number of control points at
device-chosen positions, the window path a fixed 256 entries — and may legitimately ship
the window path alone.

Neither path is guaranteed. Several window systems accept a ramp and ignore it, and a
compositing desktop usually does. A rebuild that must guarantee the correction applies has
to move it into a final shader pass, at which point it stops applying to anything the
engine did not draw.

## The ramp curve

**Contract** — maps an input level in `[0, 1]` to an output level in the same range, per
channel. Evaluated once per ramp entry, at whatever input positions the target asks for.

```text
FUNCTION ramp_value(x) -> real          # x in [0,1]; result in [0,1] before clamping
  exponent = 1 / gamma                  # guarded against a zero gamma
  gain     = (contrast + 1) / 2         # 1 at neutral, 0.75 .. 1.25 over the clamp range
  lift     = (brightness - 1) / 4       # 0 at neutral, -0.125 .. +0.125

  corrected = x raised to exponent      # the gamma curve itself
  RETURN 0.5 + gain * (corrected - 0.5) + lift
                                        # contrast pivots about mid-grey, so it
                                        # darkens shadows and brightens highlights
                                        # rather than scaling the whole picture
```

**Invariants**

- At `gamma = brightness = contrast = 1` this is the identity, term by term.
- Contrast pivots about **mid-grey**, not about black. That is the entire reason for the
  two constant terms; a naive multiply would behave as a second brightness control.
- Each channel's value is multiplied by its `balance` component **after** the offsets are
  folded in, so balance scales the black level too, and then clamped to the target's
  representable range. Out-of-range values are clamped, not wrapped — a bright setting
  flattens the highlights rather than folding them back to black.
- The window-system path normalizes its entry index by **255**, not by the entry count.
  The two agree only at 256 entries, which is the only size ever requested; a rebuild that
  parameterizes the length must normalize by `length - 1`.

**Notes** — In the original the formula is written out once per target with the constants
pre-multiplied into the target's own numeric range (16-bit fixed point for the window
ramp, a device-reported float range for the display output). They are the same curve. A
rebuild should evaluate in normalized floating point as written above and scale at the
end; the duplicated constants in the source are an artefact of avoiding a divide inside a
256-iteration loop on 2007 hardware.

**Balance is dead weight.** The per-channel multiplier is settable through two entry
points, is fixed at neutral by the constructor, and nothing in the engine ever calls
either setter. It was presumably a colour-temperature control that never reached the
options screen. A rebuild can drop it; it is documented here because leaving an unexplained
multiply in the curve is worse than saying it does nothing.
