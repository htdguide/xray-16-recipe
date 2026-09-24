# src/Layers/xrRender/SH_Constant.cpp

> An animated colour constant: four independent waveforms, evaluated at most once per frame, that a material pass can bind in place of a fixed colour.

**Needs** — [`SH_Constant.h`](SH_Constant.h.md) · [`xrEngine/WaveForm.h`](../../xrEngine/WaveForm.h.md)
**Used by** — [`SH_Constant.h`](SH_Constant.h.md)
**Tier floor** — T1: its four waveform records are read from and written to the shipped material library as raw byte images, so their layout is frozen.

## Purpose

Some surfaces pulse, flicker or fade on a schedule the material authors wrote down rather than a program computed. This record is that schedule: one waveform per colour channel, plus the evaluated result in both the float and the packed-integer form the device wants.

## State

```text
RECORD Constant
  name        : text                # registry key; interned like every other resource
  value_float : (real, real, real, real)
  value_packed: int (32-bit)        # the same colour packed; kept in step with value_float
  frame       : int                 # the frame number this value was computed for
  mode        : enum { programmable, waveform }
  R, G, B, A  : WaveForm            # one per channel, as a frozen byte image
```

**Invariants**

- The float and packed forms are always written together; no path sets one alone. The packed form exists because some bindings take a colour as a single word and repacking per bind would be wasteful.
- `frame` makes evaluation idempotent within a frame: a constant bound by ten passes is computed once. The test is equality with the current frame number, not "greater than", so a rebuild whose frame counter can repeat (a wrap, a reset on level load) must either widen the counter or clear these records.
- In *programmable* mode the waveforms are not evaluated at all — the value is whatever some code last assigned. The mode is not stored in the shipped file: a constant read from the material library is always a waveform constant, and programmable is what a constant becomes when the engine drives it directly.

## `Calculate`

**Contract** — evaluates the four waveforms against the engine's global time and stores the result, at most once per frame, and never in programmable mode. Pure apart from the memo. Called from the bind path, so it must be cheap.

```text
FUNCTION calculate()
  IF frame = current frame, RETURN
  frame = current frame
  IF mode is programmable, RETURN
  t = global time
  set value to (R.at(t), G.at(t), B.at(t), A.at(t))
```

**Notes** — Global time, not simulation time: like animated textures, these keep moving while the simulation is paused.

## `Load` · `Save`

**Contract** — reads or writes the four waveforms as four consecutive byte images, in channel order red, green, blue, alpha. Loading forces the mode to waveform. **Frozen**: the material library ships these bytes.

## `Similar`

**Contract** — true when two constants would animate identically: same mode and four channel-wise similar waveforms. Used by the material compiler to share one constant record between passes that asked for the same animation. Note the asymmetry with the rest of the chapter — this is the one comparison done *by value* rather than by identity, because a constant is compiled from parameters, not looked up by name.

## `set_float` · `set_dword`

**Contract** — assign the value directly, keeping both forms in step; this is what puts a constant into programmable use. No side effects.
