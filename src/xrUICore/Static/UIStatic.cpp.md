# src/xrUICore/Static/UIStatic.cpp

> The widget almost everything else derives from — a window that may carry a texture, a text block, a colour animation, a transform animation and a delayed hint, in a fixed draw order.

**Needs** — [`UIStatic.h`](UIStatic.h.md) · [`UIStaticItem.h`](UIStaticItem.h.md) · [`UILanimController.h`](UILanimController.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`Buttons/UIBtnHint.h`](../Buttons/UIBtnHint.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`UIStatic.h`](UIStatic.h.md)
**Tier floor** — T2: it owns device-facing material handles and is on the per-frame draw
path, but every decision here is layout and state, not bytes.

## Purpose

If a widget in this engine shows anything at all, it is probably a static or derives from
one. The file settles what "showing something" means: a picture, a text block, or both; in
what order they are drawn relative to children; how the rectangle relates to the art; and how
an authored animation curve can drive colour, rotation or scale.

The draw order it fixes — **texture, then children, then text** — is load-bearing and
surprising. A static's own text draws *over* its children, so a button's caption is never
hidden by a decoration attached to it. A rebuild that draws text with the rest of the widget
will get the shipped screens subtly wrong.

## State

```text
RECORD Static EXTENDS Window
  text          : optional<TextBlock>   # created on first use, not on construction
  item          : StaticItem            # the drawable quad: material, rect, size, colour
  texture_offset: vec2                  # shifts the art within the widget's rectangle
  stretch       : bool                  # scale art to the rectangle, or draw at authored size
  texture_on    : bool                  # draw the art at all
  heading       : bool                  # this widget may be rotated
  const_heading : bool                  # the angle is authored, not driven
  angle         : real                  # radians
  hint          : text                  # shown after a dwell; empty means none
  xform_anim    : { curve, start_time, delay, flags, original_size }
  # colour animation lives in the mixed-in controller; see UILanimController.h
```

**Invariants**

- The text block is created lazily. A static with no text allocates no text machinery, which
  matters because a busy screen has hundreds of purely decorative statics.
- `original_size` for the transform animation is captured when the animation is *assigned*,
  not when it starts, and is what the animation scales relative to and restores to when it
  ends. Assigning an animation after a resize therefore anchors it at the new size.
- The text block's own size is kept equal to the widget's size, and re-laying-out is triggered
  by noticing they have diverged at draw time — not by the resize itself.

## `Draw`

**Contract** — draws the widget's texture, then its children, then its text. Does not clip.

```text
FUNCTION draw(s)
  draw_texture(s)
  draw children in attach order      # the base window's behaviour
  draw_text(s)
```

## `DrawTexture`

**Contract** — positions and sizes the drawable quad from the widget's absolute rectangle,
then emits it, rotated if heading is on. Does nothing when the texture is off or no material
was ever created.

```text
FUNCTION draw_texture(s)
  IF NOT s.texture_on OR s has no material THEN RETURN
  r = absolute_rect(s)
  s.item.position = r.top_left + s.texture_offset

  IF s.stretch
    IF s.heading AND the art is pinned at its top-left while rotating
      swap r's width and height            # the rotated art occupies the transposed box
    s.item.size = (r.width, r.height)
  ELSE
    IF s.heading THEN swap r's width and height
    s.item.size = the authored sub-rectangle's extent     # ignore the widget's size

  IF s.heading THEN emit rotated by s.angle ELSE emit unrotated
```

**Invariants** — the width/height swap before a rotation is what keeps a rotated
non-square image inside the widget. In the stretch case it is applied only when the art is
pinned at its top-left corner during rotation, because a centred rotation needs no
transposition; in the non-stretch case it is applied unconditionally, and then discarded,
since the size comes from the art. That second swap is dead. A rebuild does the transposition
once, in the pinned case only.

**Notes** — the non-stretch branch is how an icon keeps its authored pixel size inside a
larger widget. Combined with the texture offset, that is the whole of the sprite-placement
model.

## `DrawText`

**Contract** — re-lays-out the text if the widget has been resized since the last layout, then
draws the block at the widget's absolute position. Also draws the shared hint window if this
widget currently owns it.

```text
FUNCTION draw_text(s)
  IF s has a text block
    IF s.text.size != s.size
      s.text.size = s.size; re-wrap the text
    draw s.text at absolute_position(s)
  IF the shared hint belongs to s THEN draw the hint
```

**Invariants** — the hint is a **single shared window**, not one per widget. Ownership is
claimed in `Update` and released on hover loss, and only the owner draws it. That is how the
engine guarantees at most one tooltip on screen without any coordination between widgets.

## `Update`

**Contract** — advances the colour animation, advances the transform animation, and manages
hint ownership. Runs once per frame for every shown static.

```text
FUNCTION update(s)
  update children and hover state      # the base window's behaviour
  advance the colour animation          # drives text and/or texture colour

  IF s.xform_anim.curve EXISTS
    IF the animation has not started THEN start it now
    t = seconds since start
    IF the animation is cyclic OR t * time_factor < curve length
      c = curve value at (t / time_factor)
      # the curve's colour channels are REINTERPRETED as a transform:
      force heading on; s.angle = full_turn * c.alpha / 255
      scale  = c.red / 64
      s.size = s.xform_anim.original_size * scale
    ELSE
      restore heading to its authored value and the size to the original

  IF s is hovered AND s has a hint AND nobody owns the shared hint
     AND now > s.hover_start + HINT_DELAY            # 700 ms
    claim the hint, set its text
    place it near the cursor: try above, then above-left, then below-left,
        then below-right cleared by the cursor glyph height — first that fits the canvas
```

**Invariants**

- The transform animation reads an authored **colour** curve and reinterprets its channels:
  alpha becomes an angle over a full turn, red becomes a scale over a 64-unit range. That is
  a deliberate reuse of the engine's light-animation library so that UI motion is authored
  with the same tool as light flicker — there is no separate UI animation format.
- The colour animation is measured against *continual* time (which does not stop when the
  game is paused) while the transform animation is measured against *global* time scaled by
  the game's time factor. So a pulsing colour keeps pulsing in a menu, while a spinning icon
  follows game time. Both behaviours are relied on.
- The hint appears only after 700 ms of continuous hover and only if no other widget already
  owns the shared hint — first claimant wins for as long as it stays hovered.

**Notes** — the scale divisor of 64 and the delay of 700 ms are both bare constants with no
derivation in the source. The divisor means a red channel of 64 is unity scale and 255 is
roughly four times size; the delay is a conventional tooltip dwell.

## `OnFocusLost`

**Contract** — when the cursor leaves, release the shared hint if this widget owns it. This
is the only release path, which means a widget that is hidden while hovered keeps the hint
until something else claims it — a small leak in behaviour a rebuild should close.

## `InitTexture` / `InitTextureEx` / `CreateShader` / `SetShader`

**Contract** — resolve an icon name through the registry into this widget's drawable item,
sizing the item to the icon and placing it at the widget's current position. The material
pass name is first passed through the renderer, which may substitute a backend-specific
variant. Returns whether the name was a registered icon (as opposed to a raw texture path);
see [`UITextureMaster.cpp`](../XML/UITextureMaster.cpp.md).

## `AdjustHeightToText` / `AdjustWidthToText`

**Contract** — resize the widget to fit its text. Height: set the text block's width to the
widget's width, re-wrap, and take the resulting visible height. Width: measure the string in
the current font, convert that pixel measurement back into canvas units, and take it as the
width.

**Invariants** — the two are asymmetric on purpose. Height-fitting assumes the width is fixed
and the text wraps; width-fitting assumes a single unwrapped line. A list of wrapped text rows
uses the first (see [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)); a context menu sizing
itself to its longest entry uses the second.

## `SetXformLightAnim` / `ResetXformAnimation`

**Contract** — attach a named curve from the light-animation library to this widget's
transform, capturing the current size as the animation's reference. Naming nothing detaches.
Resetting restamps the start time to now.

## `ColorAnimationSetTextureColor` / `ColorAnimationSetTextColor`

**Contract** — the two sinks the colour animation drives. Each either replaces the whole
colour or substitutes only the alpha channel, according to the animation's alpha-only flag.
Which sinks are driven is decided by the flags read from the layout.

## `TextItemControl`

**Contract** — returns the text block, creating it on first call with left alignment as the
default. Every text operation on a static goes through here, which is why a static that is
asked for its text at all acquires a block even if it never shows one.

## Debug inspector

**Contract** — exposes the texture flags, the texture offset, the heading flags and angle, and
the hint text, on top of the base window's sheet. The drawable item's and the text block's own
sheets are marked as not yet written.
