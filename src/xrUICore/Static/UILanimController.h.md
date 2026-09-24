# src/xrUICore/Static/UILanimController.h

> Lets any widget be driven by a named curve from the engine's light-animation library, reinterpreting that curve's colour channels as whatever the widget needs — text colour, texture colour, or both, whole or alpha only.

**Needs** — [`UILanimController.cpp`](UILanimController.cpp.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`UILanimController.cpp`](UILanimController.cpp.md) · [`UIStatic.cpp`](UIStatic.cpp.md) · [`UIStatic.h`](UIStatic.h.md)
**Tier floor** — T3: a time-driven table lookup and a fan-out to two sinks.

## Purpose

The engine already has an authoring tool and a data format for time-varying colour: the
light-animation library, used for flickering lamps and pulsing world effects. Rather than
inventing a UI animation format, the interface borrows it. A layout names a curve, says which
of the widget's colour channels it drives, and the widget pulses.

This header is the substance holder for that mechanism — the implementation is inline here
and the companion source carries only the container variant — so it is written in full.

The design's one real decision is the **sink split**: the curve does not write a colour
anywhere itself, it calls two overridable operations, "apply to texture colour" and "apply to
text colour". A widget decides what those mean. A plain static assigns them; a container
forwards them to its children; a widget with several textures could spread them. That is why
one animation mechanism serves the whole toolkit.

## State

```text
CONSTANT CYCLIC        = flag 0   # loop forever instead of stopping at the curve's end
CONSTANT ONLY_ALPHA    = flag 1   # take only the curve's alpha, keep the widget's own hue
CONSTANT TEXT_COLOR    = flag 2   # drive the text
CONSTANT TEXTURE_COLOR = flag 3   # drive the texture

RECORD ColorAnimation
  curve      : optional<Curve>   # borrowed from the shared library, never owned
  start_time : real (seconds)    # negative means "not started yet"
  delay      : real (seconds)    # wait this long after the start before anything moves
  flags      : the four above

RECORD TransformAnimation EXTENDS ColorAnimation
  original_size : vec2           # what a scale is relative to and restores to
```

**Invariants**

- The curve is **borrowed** from the process-wide library. Nothing here owns or releases it,
  and detaching is just clearing the reference.
- A negative start time means "start on the next update". That is how an animation assigned
  at load time begins when the screen is first shown rather than when the document was read.
- An animation must drive at least one of the two sinks. Assigning a curve with neither flag
  set is asserted against — a curve driving nothing is always an authoring mistake.
- Time is measured in *continual* seconds, which do not stop when the game is paused. A menu's
  pulsing highlight therefore keeps pulsing.

## `CUILightAnimColorConroller`

**Contract** — the interface. Three demands and two optional sinks:

```text
INTERFACE ColorAnimated
  is_color_animation_present() -> bool     # is an animation attached and still running
  reset_color_animation()                  # restamp the start time to now (+ delay)
  set_color_animation(name, flags, delay)  # attach by name, or detach when unnamed
  apply_texture_color(colour, alpha_only)  # sink; default: ignore
  apply_text_color(colour, alpha_only)     # sink; default: ignore
```

**Notes** — the sinks default to doing nothing, so a widget may opt into one, both or
neither. That is how the interface can be mixed into a widget that has no text.

## `CUILightAnimColorConrollerImpl`

**Contract** — the reusable implementation, mixed into any widget that wants it. Holds one
colour animation and drives the sinks from it.

```text
FUNCTION set_color_animation(a, name, flags, delay)
  IF name is empty THEN a.curve = none; RETURN
  a.curve = the library's curve named `name`      # absent name yields nothing
  a.delay = delay
  a.flags = flags
  assert a.curve is absent OR at least one of TEXT_COLOR / TEXTURE_COLOR is set

FUNCTION reset_color_animation(a)
  a.start_time = now + a.delay

FUNCTION is_color_animation_present(a) -> bool
  IF a.curve is absent THEN RETURN false
  IF a.flags has CYCLIC OR a.start_time < 0 THEN RETURN true
  RETURN now - a.start_time < curve length

FUNCTION update_color_animation(a)
  IF a.curve is absent THEN RETURN
  IF a.start_time < 0 THEN reset_color_animation(a)
  IF now < a.start_time THEN RETURN                 # still inside the delay
  IF a.flags has CYCLIC OR now - a.start_time < curve length
    c = curve value at (now - a.start_time)         # the library wraps for cyclic curves
    IF a.flags has TEXTURE_COLOR
      apply_texture_color(a.flags has ONLY_ALPHA ? alpha of c : c, only_alpha)
    IF a.flags has TEXT_COLOR
      apply_text_color(a.flags has ONLY_ALPHA ? alpha of c : c, only_alpha)
```

**Invariants**

- The delay is stored in milliseconds by the caller and divided into seconds when stamping
  the start time — two different units for one field. A rebuild picks one.
- When the alpha-only flag is set, the sink receives the *alpha channel as the whole value*
  and a flag saying so; the sink is then responsible for substituting it into the widget's
  existing colour. That is why the sinks take both a colour and a flag rather than just a
  colour.
- A non-cyclic animation that has run past its end simply stops driving the sinks; it does
  **not** restore the widget's original colour. Whatever the last frame wrote stays. That is
  relied on by fade-ins, and it is why the transform animation — which *does* restore — has
  to do the restoring itself.

## `CUIColorAnimConrollerContainer`

**Contract** — a window that has no colour of its own and forwards the animation to its
children: the texture sink reaches every child that is a texture owner, and the text sink
reaches every child that is itself colour-animated. Implemented in
[`UILanimController.cpp`](UILanimController.cpp.md).

**Notes** — this is how one authored curve pulses a whole group. The fan-out is one level
deep for textures and recursive for text, because the text sink forwards to another
controller which forwards again. That asymmetry is unexplained in the source and looks
accidental; a rebuild should make both levels behave the same way.
