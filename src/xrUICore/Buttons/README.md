# src/xrUICore/Buttons — the press machine and its four dressings

> One state machine turns a stream of pointer events into exactly one click notification.
> Everything above it is a choice of how the four states look.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The **button** is a static with a press state machine, a set of keyboard accelerators, and the
hover-hint behaviour every derived button inherits. Its job is not "look pressed" — it is to
guarantee that a press, a drag off the widget, a drag back on and a release produce one click
and not two, and that a release outside produces none.

The **four-state button** is the one the shipped games actually use. Rather than drawing a
pressed look by offsetting the label, it picks one of four background textures and one of four
text colours from the enabled/pressed/hovered condition each frame, and plays the hover and
click sounds.

The **check box** is a latching button: its checked state *is* its press state. It notifies
set and reset as distinct messages, can drive a dependent control's enabled flag, and is
itself a settings control bound to one boolean console variable.

The **radio button** is a tab button wearing a fixed radio graphic — mutual exclusion comes
free from the tab group rather than being implemented again — which announces the group's
change as a radio selection.

The **hint box** is the process-wide hover hint any widget may borrow for a frame.

## The load-bearing ideas

**One press, one click.** The state machine is the load-bearing part of the directory. A click
is emitted on release *inside* the widget after a press inside it; leaving while held and
returning must not emit two; releasing outside must emit none. Capture is what makes this
possible — the button captures on press, so it keeps receiving movement after the pointer
leaves.

**Visual state is derived, not stored.** The four-state button recomputes its background and
text colour from (enabled, pressed, hovered) every frame rather than switching on transitions.
A rebuild that drives it from transitions will drift out of sync the first time a state changes
without a pointer event — a screen disabling a button, for instance.

**Mutual exclusion is delegated.** The radio button inherits it rather than implementing it,
which means a group of radio buttons *is* a tab control; if a rebuild separates them it must
reimplement exclusivity twice.

**The hint box is shared and claimed.** There are two process-wide hint boxes, built from one
of two XML layouts depending on which game's data is mounted, and exactly one widget owns each
at a time. They are drawn after the rest of the interface, with the cursor.

**A check box is simultaneously a widget and a settings binding.** That coupling is imposed by
the XML vocabulary, which declares the console variable in the same element as the geometry.

## The twins

| Twin | Role |
|---|---|
| [`UIButton.cpp`](UIButton.cpp.md) | The press state machine, accelerators, and the hover-hint behaviour every button inherits |
| [`UIButton.h`](UIButton.h.md) | The clickable widget's declaration |
| [`UI3tButton.cpp`](UI3tButton.cpp.md) | Mapping (enabled, pressed, hovered) onto one of four backgrounds and four text colours each frame; hover and click sounds |
| [`UI3tButton.h`](UI3tButton.h.md) | The button the shipped games use |
| [`UICheckButton.cpp`](UICheckButton.cpp.md) | The latch: checked state as press state, separate set/reset notifications, a dependent control's enabled flag, one boolean console variable |
| [`UICheckButton.h`](UICheckButton.h.md) | The check box's declaration |
| [`UIRadioButton.cpp`](UIRadioButton.cpp.md) | Pinning the radio graphic, indenting the label past it, re-announcing the tab change as a radio selection |
| [`UIRadioButton.h`](UIRadioButton.h.md) | The radio button as a tab button with a fixed graphic |
| [`UIBtnHint.cpp`](UIBtnHint.cpp.md) | The shared hint box: two layouts depending on the mounted game, claimed by one widget at a time, drawn last |
| [`UIBtnHint.h`](UIBtnHint.h.md) | The two process-wide hover-hint boxes |
