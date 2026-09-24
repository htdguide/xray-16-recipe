# src/editors/xrWeatherEditor/property_color_base.cpp

> A colour as three linked real rows plus one row that shows and parses all three — the editor's most-used property type, and the one composite binding in the directory.

**Needs** — [`property_color_base.hpp`](property_color_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_float_limited.hpp`](property_float_limited.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md) · [`property_converter_color.hpp`](property_converter_color.hpp.md) · [`xrSdkControls/Controls/Interfaces/IIncrementable.cs`](../xrSdkControls/Controls/Interfaces/IIncrementable.cs.md) · [`xrSdkControls/Controls/Interfaces/IMouseListener.cs`](../xrSdkControls/Controls/Interfaces/IMouseListener.cs.md)
**Used by** — [`property_color.cpp`](property_color.cpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md) · [`property_color_reference.cpp`](property_color_reference.cpp.md)
**Tier floor** — T1: it declares a three-real record with a fixed layout that is passed by value across a boundary between two separately compiled modules.

## Purpose

Most of what a weather keyframe holds is colour: ambient, hemisphere, sky, fog, sun, thunderbolt, lens flare. This is how one is edited, and it is the only property in the directory that is *composite* — a value that both has a one-line representation and expands into sub-rows.

The design has three parts and each is a decision:

- the colour expands into **three independent real rows**, each clamped to the unit interval and each carrying its own drag-scrub;
- the colour's own row renders as three formatted numbers and **parses them back**, so an author can type or paste a whole colour;
- the whole colour responds to a drag by **moving all three channels together**, which is a brightness gesture.

## State

```text
RECORD ColorBinding
  container   : PropertyContainer     # holds the three component rows
  components  : ChannelAccessors      # getter/setter pairs that read-modify-write the colour
  attributes  : list<Attribute>       # the caller's, plus "notify my parent on change"

RECORD Color                          # layout declared, not inferred — it crosses the boundary
  r, g, b : real
```

**Invariants** — the binding stores no colour. Every read goes through the abstract accessor a subclass supplies ([callable-bound](property_color.cpp.md) or [field-bound](property_color_reference.cpp.md)), so the three component rows, the composite row and the engine can never disagree.

**Notes** — the three-real record is declared here rather than reusing the engine's own colour type, and the reason is the boundary: see [`property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md). The engine's colour carries alignment chosen for wide-float math and cannot cross safely.

## The component accessors

**Contract** — each of the three channels is exposed as a getter/setter pair. A getter reads the whole colour and returns one channel; a setter reads the whole colour, replaces one channel, and writes the whole colour back.

```text
FUNCTION set_channel(which, value)
  current = read_color()        # read-modify-write: there is no per-channel write
  current[which] = value
  write_color(current)
```

**Invariants** — read-modify-write is forced by the engine's accessor being whole-colour, and it is also correct: a colour written by the editor is always a complete colour, so a half-applied edit cannot be observed by the running weather.

**Notes** — the accessors live in a small unmanaged object holding a collector-safe reference back to the managed binding. That double indirection is the boundary again: the property description needs unmanaged callables, and they need to reach a managed object.

## The three component rows

**Contract** — at construction the binding fills its own container with three rows named red, green and blue, in a category called `components`, each a [real clamped to 0..1](property_float_limited.cpp.md) with a fixed drag step, each rendered by [the real formatter](property_converter_float.cpp.md), and each marked so that editing it notifies the parent row.

**Invariants** — the three rows are added in red, green, blue order, and [the colour renderer](property_converter_color.cpp.md) asserts that exactly three exist and re-sorts them into that order. Colour channels have no alphabetical order worth having, so the order is asserted rather than derived.

The "notify my parent on change" marking is what makes the collapsed colour row update when a component is edited; without it the author would edit green and see the summary line stay stale.

**Notes** — the per-component drag step is a small fixed fraction of the unit range. It is the step for typing and for the spin buttons, not for the whole-colour drag below, which uses its own factor.

## `get` / `set` as a grid value

**Contract** — reading yields the container, which is what makes the row expandable. Writing takes a three-real value, narrows it, and writes the whole colour.

## `increment` — the brightness drag

**Contract** — a horizontal drag over the colour row adds the same amount to all three channels, each clamped to the unit interval independently.

```text
FUNCTION increment(pixels : real)
  step = pixels * DRAG_FACTOR
  colour = read_color()
  FOR EACH channel IN (r, g, b)
    channel = clamp(channel + step, 0, 1)
  write_color(colour)
```

**Notes** — two decisions here.

**All three channels move by the same absolute amount**, not by the same ratio. So the gesture is an exposure shift, not a brightness multiply, and dragging a saturated colour towards white *desaturates* it — the channels converge as they clamp. Whether that was intended is not recoverable; it is what an author gets, and it is arguably the more useful of the two for weather authoring, where the ambient term is a level shift.

**Clamping is per channel, and the clamp is not backed out.** Once one channel saturates, continuing to drag moves only the others, and dragging back does not restore the original ratio. The colour is not recoverable by reversing the gesture. A rebuild that wants a reversible drag must clamp the *step*, not the result.

The drag factor — the value units per pixel — is a small constant. Nothing explains its magnitude; it is a feel constant, tuned by hand.

## `on_double_click`

**Contract** — nothing. The whole body is disabled in the source.

**Notes** — the disabled code opened the platform's colour dialog, converted the unit-interval colour to and from byte channels, and refreshed the grid. It is exactly the feature a colour property most wants, and it is switched off.

The likely reason is visible in [`entry_point.cpp`](entry_point.cpp.md): a modal dialog starves the engine's idle pump, so opening a colour picker freezes the rendered view — which, for a tool whose entire purpose is watching the sky change as you edit, defeats the point. The editor ships with three sliders and no picker instead.

A rebuild should offer a *modeless* picker, or draw one inside the engine's own overlay, and it should note that [`ColorPicker`](../xrSdkControls/Controls/ColorPicker/ColorPicker.cs.md) — a complete, live-updating, alpha-aware picker — already exists in the control library and is not used here. Connecting the two is the single highest-value change a rebuild of this editor could make.
