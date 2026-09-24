# src/xrEngine/WaveForm.h

> A five-shape periodic function with amplitude, offset, phase and frequency — how authored data says "oscillate this".

**Needs** — _none_
**Used by** — [`SH_Constant.cpp`](../Layers/xrRender/SH_Constant.cpp.md) · [`SH_Constant.h`](../Layers/xrRender/SH_Constant.h.md) · [`SH_Matrix.cpp`](../Layers/xrRender/SH_Matrix.cpp.md) · [`SH_Matrix.h`](../Layers/xrRender/SH_Matrix.h.md) · [`PropertiesListTypes.h`](../xrServerEntities/PropertiesListTypes.h.md)
**Tier floor** — T1: a 20-byte record with a frozen layout, embedded in material descriptions read as memory images

## Purpose

Material descriptions, texture-coordinate animations and light properties in the shipped
data all need to say "vary this value over time". Rather than a scripting hook, the format
stores a *waveform*: which of five periodic shapes, plus four numbers that scale and shift
it. This file is that record and its evaluation.

It is header-only with no implementation file because it is a value type evaluated in the
inner loops of material recording — every animated texture coordinate calls it per pass
per frame, so a call boundary would cost more than the arithmetic.

## State

```text
RECORD WaveForm                 # packed on a 4-byte boundary; layout is frozen
  function : int (32-bit)       # one of the shapes below
  arg      : real[4]            # [offset, amplitude, phase, frequency]
                                # defaults: 0, 1, 0, 1  -> "unit sine at 1 Hz"

ENUM Shape
  CONSTANT = 0    # always zero: the waveform contributes only its offset
  SIN
  TRIANGLE
  SQUARE
  SAWTOOTH
  INVSAWTOOTH
```

The four arguments are positional and their meaning is fixed by the evaluation below; the
authoring tools wrote them in this order and nothing names them in the file.

## `Calculate`

**Contract** — evaluates the waveform at a time in seconds. Pure, allocation-free, total:
every shape is defined everywhere. The shape function is evaluated on the *fractional part*
of the scaled, shifted time, so each shape only has to be correct on the unit interval and
periodicity comes from the wrapping rather than from the shape.

```text
FUNCTION calculate(w, t) -> real
  x = (t + w.arg[2]) * w.arg[3]        # phase shift, then frequency scale
  RETURN w.arg[0] + w.arg[1] * shape(w.function, x - floor(x))
```

**Invariants** — `shape` is called only with an argument in [0, 1); every shape returns a
value in [-1, 1], so the output is bounded by offset ± amplitude. A rebuild that changes a
shape's range changes the meaning of every amplitude in the shipped data.

## The shapes

**Contract** — each maps the unit interval to [-1, 1] with one full period.

```text
FUNCTION shape(kind, t) -> real       # t in [0, 1)
  CONSTANT     -> 0
  SIN          -> sin(t * 2pi)
  TRIANGLE     -> arcsin(sin((t - 0.25) * 2pi)) / (pi/2)
  SQUARE       -> sign(cos(t * pi))
  SAWTOOTH     -> arctan(tan((t + 0.5) * pi)) / (pi/2)
  INVSAWTOOTH  -> -SAWTOOTH
```

**Notes** — triangle and sawtooth are built out of inverse trigonometry rather than the
obvious piecewise arithmetic. That is not an optimization; it is how the original is
written, and it matters for two reasons. First, the shapes it produces are exact matches
for the piecewise forms, so a rebuild may substitute the cheap versions. Second, both
expressions are *singular* at the points where the underlying tangent blows up — the
sawtooth's discontinuity is reached as an infinity that the floating-point division then
resolves — so a rebuild using piecewise arithmetic must make sure it picks the same side
of the discontinuity, or every sawtooth-animated texture jumps one frame differently.
`CONSTANT` deliberately returns zero, not one: a constant waveform contributes its offset
and nothing else.

The sign helper is written as a division by magnitude, which is undefined at exactly zero.
It is only ever fed a cosine, which is zero at two points of the period; the resulting
behaviour at those two instants is whatever the platform's division produces. Nothing
depends on it.

## `Similar`

**Contract** — tests whether two waveforms are close enough to be treated as the same, used
to merge material passes that would otherwise differ only in an animation nobody can see.
Compares offset and amplitude first; if the amplitude is effectively zero, the waveforms
are equal regardless of shape, phase or frequency — a zero-amplitude oscillation is just
its offset. Otherwise shape, phase and frequency must all match within tolerance.

```text
FUNCTION similar(a, b) -> bool
  IF a.arg[0] NOT NEAR b.arg[0]   RETURN false     # offset
  IF a.arg[1] NOT NEAR b.arg[1]   RETURN false     # amplitude
  IF a.arg[1] IS NEAR zero        RETURN true      # amplitude 0: the rest cannot matter
  IF a.function != b.function     RETURN false
  IF a.arg[2] NOT NEAR b.arg[2]   RETURN false     # phase
  IF a.arg[3] NOT NEAR b.arg[3]   RETURN false     # frequency
  RETURN true
```

**Notes** — the early return on zero amplitude is the load-bearing line: without it, the
material system would keep separate passes for a dozen "disabled" animations that all
evaluate identically.
