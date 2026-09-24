# src/xrUICore/TrackBar — the slider

> A draggable thumb on a four-state background, reporting either an integer or a real value,
> and bound to a console variable.

Part of [chapter 15](../README.md).

## What this directory is responsible for

One control: a horizontal track with a button as its thumb. The thumb's position maps linearly
onto a range, the range is either integer or real, and the widget is a settings control bound
to one console variable. An optional label beside it displays the value through a format
string from the layout.

It can also be run as a boolean, in which case the thumb has two positions and the control is a
check box wearing a slider's clothes — which is how the shipped options screens present a few
on/off settings.

## The load-bearing ideas

**Integer and real are one control with two interpretations of the same storage**, not two
controls. The bounds, step, value and backup are held once and read as whichever kind the
widget was configured for. A rebuild with a tagged value type expresses this directly; what
must survive is that an integer setting never round-trips through a real, because the shipped
configuration expects exact integers back.

**Dragging is a capture, and so is keyboard stepping.** The thumb captures the pointer on press
so the drag survives the cursor leaving the track; arrow keys and a gamepad axis step by the
configured step without any capture. Both paths converge on one position-update routine, so the
value can never be set two ways.

**Inversion is authored.** A track can be declared inverted, so that the visual left end is the
numeric maximum. This exists because a few shipped settings read naturally in the opposite
direction, and it is a property of the layout, not of the value.

**The bounds may arrive after construction.** A flag records whether they have been set, because
the layout and the console variable both want to supply them and whichever comes second must
not overwrite the first. That ordering is the fiddly part of the control.

## The twins

| Twin | Role |
|---|---|
| [`UITrackBar.cpp`](UITrackBar.cpp.md) | The thumb's drag capture, the keyboard and controller stepping, the position-to-value map, inversion, the boolean mode, and the settings protocol |
| [`UITrackBar.h`](UITrackBar.h.md) | Its declaration, over a four-state background |
