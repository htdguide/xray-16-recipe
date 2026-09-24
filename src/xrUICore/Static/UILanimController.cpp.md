# src/xrUICore/Static/UILanimController.cpp

> Broadcasts one colour-animation sample down a subtree, so a whole panel of widgets pulses as one under a single light animation.

**Needs** — [`UILanimController.h`](UILanimController.h.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md)
**Used by** — [`UILanimController.h`](UILanimController.h.md)
**Tier floor** — T3: a tree walk pushing a colour into whichever children accept one.

## Purpose

A *light animation* is an authored colour curve the engine shares between world lights and the
UI — the pulsing red of a low-health warning, the flicker of a damaged panel. An individual
widget can subscribe to one directly. This file supplies the other case: a container that
subscribes once and pushes each sample onto everything it holds, so an entire framed panel
throbs in step rather than each part animating independently.

The split matters because the two sinks are different. A widget's *texture* colour and its
*text* colour are separate channels, and the container must reach them through different
routes: any child that owns a texture can take the first directly, but only a child that is
itself a colour-animation container can be trusted with the second.

## State

`Stateless.` The animation, its phase and the current sample belong to the colour-animation
mixin this container extends; this file only distributes.

## `Update`

**Contract** — per frame: advance the window tree as usual, then sample the colour animation
and push it, which is what makes the broadcast happen once per frame rather than on demand.

## `ColorAnimationSetTextureColor`

**Contract** — the texture channel. Offers the sample to every immediate child that owns a
texture, writing the child's texture colour directly; children with no texture are skipped.
In alpha-only mode the child keeps its own colour and receives only the new opacity;
otherwise the whole colour is replaced.

```text
FUNCTION color_animation_set_texture_color(colour, only_alpha)
  FOR EACH child IN children
    IF child owns a texture
      child.texture_colour <- only_alpha
        ? child.texture_colour with colour's alpha substituted
        : colour
```

**Notes** — alpha-only mode is the common one: it lets an authored curve fade a panel in and
out without discarding the per-widget tinting a layout applied. A rebuild needs both modes.

This channel writes the child's colour *field*; it does not ask the child to distribute
anything. So it stops one level deep, and a nested container — which owns no texture of its
own — is skipped entirely, taking its whole subtree with it.

## `ColorAnimationSetTextColor`

**Contract** — the text channel. Hands the sample to every immediate child that participates
in colour animation at all, letting each decide what to do with it. A plain picture-and-text
widget paints its own text; a nested container re-broadcasts to its own children.

**Notes** — the asymmetry with the texture channel is the decision worth carrying. Text is
distributed by *asking* and therefore recurses to any depth; texture colour is distributed by
*writing* and therefore reaches exactly one level. A panel whose art is a direct child and
whose labels are nested inside sub-panels — which is how the shipped screens are built — gets
both effects; the same panel restructured gets only the text one.

A rebuild is free to make both channels recurse by asking, and should, but must then check
that no shipped screen relied on a nested picture *not* being tinted.
