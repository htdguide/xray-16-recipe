# src/xrUICore/Buttons/UI3tButton.cpp

> Maps the button's enabled/pressed/hovered condition onto one of four background textures and one of four text colours each frame, and plays the hover and click sounds.

**Needs** — [`UI3tButton.h`](UI3tButton.h.md) · [`UIButton.h`](UIButton.h.md) · [`InteractiveBackground/UI_IB_Static.h`](../InteractiveBackground/UI_IB_Static.h.md) · [`Hint/UIHint.h`](../Hint/UIHint.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`UI3tButton.h`](UI3tButton.h.md)
**Tier floor** — T3: a four-way selection per frame plus two fire-and-forget 2D sounds.

## Purpose

The plain button expresses "pressed" by shifting its texture a pixel. That is not how the
game looks. This file replaces that with a *visual state*: a set of four backgrounds, one of
which is current, chosen every frame from the button's condition. The choice is recomputed
rather than driven by transitions, which means the button is always consistent with its
enabled flag and its press state even when those are changed behind its back — by an options
screen enabling a control, say.

## State

```text
RECORD ThreeTexButton EXTENDS Button
  background      : optional<InteractiveBackground>     # quad form
  back_frameline  : optional<InteractiveBackground>     # stretched three-segment form
  frameline_mode  : bool          # which of the two is created by init
  vertical        : bool          # orientation of the frameline form
  text_color      : list<colour>  # indexed by state: enabled, disabled, highlighted, touched
  use_text_color  : list<bool>    # per state; the enabled entry is never consulted
  sound_hover     : sound handle
  sound_click     : sound handle
```

**Invariants**

- Exactly one of the two background forms exists after initialization.
- The enabled colour is the fallback for every state whose "use" flag is clear. Defaults:
  enabled and highlighted and touched are opaque white, disabled is opaque light grey, and
  only *disabled* starts with its flag set — so an unconfigured button dims when disabled and
  does nothing else.
- Texture drawing is disabled until a texture set has been supplied; the four-state machinery
  is inert before that and the button draws only its text.

## `init_button`

**Contract** — creates the background child (one kind or the other, per the frameline flag),
sizes it to the whole button rect at the origin, and then sets the button's own position and
size. Idempotent: a second call re-sizes the existing background rather than creating another.

**Notes** — the background is an ordinary child window with auto-delete, so it participates
in the normal draw and hit-test walk. It is placed at the child origin, which is why every
`SetWidth`/`SetHeight` override must forward the new size to it — a background that does not
track the button's rect is the most common bug this class invites.

## `init_texture`

**Contract** — two forms. The one-name form derives four names by appending `_e`, `_d`, `_t`
and `_h` and delegates. The four-name form asks the background to load a texture into each of
the four slots, records whether any of them failed, and enables texture drawing regardless.
The failure is reported to the caller but never fatal here — the `fatal` flag is passed
through to the texture lookup, which decides whether a missing entry aborts.

**Invariants** — after either form, texture drawing is on. The loader sets the "current" slot
to whichever state it loaded last, so the very first frame may show the highlighted texture;
the per-frame update immediately corrects it.

## `update`

**Contract** — per-frame, recomputes both the current background state and the text colour
from the same priority ladder.

```text
FUNCTION update()
  inherited.update()

  # one ladder, applied twice: highest precedence first
  state <- IF NOT enabled            THEN disabled
           ELSE IF press_state == pushed THEN touched
           ELSE IF cursor_over_window    THEN highlighted
           ELSE                              enabled

  IF texture_enabled THEN background.current <- state

  text.colour <- IF use_text_color[state] THEN text_color[state]
                                          ELSE text_color[enabled]
```

**Notes** — disabled outranks pressed, and pressed outranks hovered. That ordering is what
makes a latching check button stay visibly checked while the pointer is over it, and what
makes a disabled control look disabled even if it was left in a pressed state.

## `draw_texture`

**Contract** — draws the current background. The quad form is forced to stretch to the widget
rect on every draw; the stretched-line form manages its own segment geometry. Draws nothing
when no texture set was supplied.

## `on_click` / `on_focus_receive`

**Contract** — `on_click` fires the inherited click notification and then plays the click
sound; `on_focus_receive` — which here means "the pointer entered" — plays the hover sound.
Both are silent when the corresponding sound was never loaded. Both sounds are played in 2D
with no attenuation and are not tracked, so a rapid re-hover overlaps rather than restarts.

## `set_width` / `set_height`

**Contract** — resize the button and forward the new dimension to whichever background exists.

## `set_texture_offset`

**Contract** — shifts the texture inside every state of the quad background. Silently does
nothing in stretched-line mode, which has no single texture to offset.
