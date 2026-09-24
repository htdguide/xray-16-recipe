# src/xrUICore/uiabstract.h

> The three small vocabularies every widget shares — the texture-owner interface, the alignment enumerations, and the notion of being selected.

**Needs** — [`xrEngine/GameFont.h`](../xrEngine/GameFont.h.md)
**Used by** — [`UIFrameRect.h`](../xrGame/UIFrameRect.h.md) · [`UILines.h`](Lines/UILines.h.md) · [`UIListBoxItem.h`](ListBox/UIListBoxItem.h.md) · [`UIWindow.cpp`](Windows/UIWindow.cpp.md) · [`UIWindow.h`](Windows/UIWindow.h.md) · [`UITextureMaster.cpp`](XML/UITextureMaster.cpp.md)
**Tier floor** — T3: an interface plus two enumerations.

## Purpose

Three unrelated things that are together only because they are needed everywhere. The one
that matters is `ITextureOwner`: it is the contract that lets XML-driven construction
([`UIXmlInitBase.cpp`](XML/UIXmlInitBase.cpp.md)) apply a `<texture>` element to a widget
without knowing what kind of widget it is. A plain picture, a nine-slice frame and a
three-part stretchable line all satisfy it and all interpret it differently.

## State

`Stateless.`

## `ITextureOwner`

**Contract** — what a widget must provide to be textured from data. Every method is a
demand on the implementor:

```text
INTERFACE TextureOwner
  init_texture(name, fatal) -> bool
      # resolve `name` through the shared texture registry; on failure either
      # abort (fatal) or report false and leave the widget untextured
  init_texture_ex(name, material_pass, fatal) -> bool
      # same, but with an explicitly named material pass instead of the default
  set_texture_rect(r) / get_texture_rect() -> Rect
      # the sub-rectangle of the atlas page, in texel units
  set_texture_color(c) / get_texture_color() -> color
      # modulation colour; also the channel a light animation drives
  set_stretch_texture(b) / get_stretch_texture() -> bool
      # true: scale the art to the widget's rectangle
      # false: draw the art at its authored size, ignoring the widget's size
```

**Invariants** — `init_texture` is `init_texture_ex` with the engine's default material
pass. An implementor that has no single rectangle (the frame window has nine, the frame line
has three) still must answer `get_texture_rect`; it returns the one a caller most plausibly
meant, and its `set_texture_rect` is a deliberate no-op. That asymmetry is honest: the
interface is shaped for the single-image case and the multi-part widgets partially ignore it.

**Notes** — "shader" in the parameter names means a *material pass description* loaded from
data, not a GPU program; see §5 of the system requirements. The default is the engine's
plain textured-quad pass.

## `EWindowAlignment`

**Contract** — how a window's position is to be interpreted: not at all (`None`), as a
top-left corner (`Left`), as a centre (`Center`), or as an offset along one free axis with
the other pinned to a canvas edge (`Right`, `Top`, `Bottom`). Consumed by the rectangle
derivation in [`UIWindow.cpp`](Windows/UIWindow.cpp.md).

**Notes** — the values are declared as distinct bits, inviting combination, but every
consumer switches on them as if they were mutually exclusive. They are exclusive. A rebuild
should declare a plain enumeration and drop the bit pattern.

## `ETextAlignment` / `EVTextAlignment`

**Contract** — horizontal text alignment is borrowed wholesale from the font layer
(left/centre/right), so that the toolkit and the font renderer cannot disagree. Vertical
alignment (top/centre/bottom) is the toolkit's own, because the font layer lays out a line
at a time and has no notion of a box to centre within.

## `CUISelectable`

**Contract** — one boolean, "am I the selected one", with an overridable setter so a widget
can repaint when it changes. Mixed into list items. It exists as a separate mixin rather
than a window field because only container-managed items have the concept, and the container
([`UIScrollView.cpp`](ScrollView/UIScrollView.cpp.md)) enforces at most one selected child by
clearing every sibling.
