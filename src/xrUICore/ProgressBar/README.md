# src/xrUICore/ProgressBar — showing a fraction

> Three shapes for one idea: a linear fill, two linear fills compared, and a ring of sectors.

Part of [chapter 15](../README.md).

## What this directory is responsible for

Turning a value in a range into something visible. The **bar** is the general one: a range, a
drawn position that chases its target rather than jumping, one of six fill directions, and an
optional colour that interpolates along the fill. The **double bar** stacks two of them in one
place to show a current value against a reference — the inventory's "this armour versus that
armour" comparison — colouring the back bar green or red according to which value is larger.
The **shape** is the circular indicator: a fan of textured triangular sectors, each sector's
opacity set by a smooth step across the progress fraction.

## The load-bearing ideas

**The drawn position lags the value deliberately.** An inertia factor eases the bar toward its
target each frame, which is what makes a health bar read as draining rather than teleporting.
It is a display property only; the value is exact the moment it is set.

**Fill is a clip, not a resize.** The fraction becomes a clip rectangle over a full-size
texture, so the fill's art does not stretch as it grows. Six directions exist because the
shipped screens fill bars from each edge and from the centre outward.

**The comparison bar decides which value is in front by magnitude, not by role.** The larger
value goes on the back bar and the smaller in front, so the visible difference is always the
back bar showing past the front one; the colour is what says whether that difference is a gain
or a loss. A rebuild that fixes "current in front, reference behind" gets the wrong picture
half the time.

**The circular indicator fades sectors rather than switching them.** Each sector's opacity is a
smooth step across the fraction, so the ring fills continuously from a small number of
segments. The sector count is a trade between smoothness and quad count.

## The twins

| Twin | Role |
|---|---|
| [`UIProgressBar.cpp`](UIProgressBar.cpp.md) | The eased position, the fraction-to-clip-rectangle conversion for six directions, and the interpolated fill colour |
| [`UIProgressBar.h`](UIProgressBar.h.md) | Its declaration |
| [`UIDoubleProgressBar.cpp`](UIDoubleProgressBar.cpp.md) | Larger value on the back bar, smaller in front, coloured by which was larger |
| [`UIDoubleProgressBar.h`](UIDoubleProgressBar.h.md) | The comparison bar's declaration |
| [`UIProgressShape.cpp`](UIProgressShape.cpp.md) | The ring: textured triangular sectors fanned from a centre, each faded by a smooth step |
| [`UIProgressShape.h`](UIProgressShape.h.md) | Its declaration |
