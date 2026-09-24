# src/xrUICore/XML/UITextureMaster.cpp

> The process-wide registry that maps a logical icon name onto (atlas page, sub-rectangle), and caches one material per (page, pass) so that a hundred widgets sharing an atlas share one material.

**Needs** — [`UITextureMaster.h`](UITextureMaster.h.md) · [`xrUIXmlParser.h`](xrUIXmlParser.h.md) · [`Static/UIStaticItem.h`](../Static/UIStaticItem.h.md) · [`uiabstract.h`](../uiabstract.h.md) · [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UITextureMaster.h`](UITextureMaster.h.md)
**Tier floor** — T2: two maps and material creation. It is off the per-frame path — every
lookup happens at screen-load time — but it holds device resources.

## Purpose

No widget names a texture file. Every widget names an *icon*, and this registry answers what
page that icon lives on and which rectangle of it to use. That indirection is what lets the
game ship a handful of large atlas pages instead of thousands of small textures, and it is
what lets a UI style override one icon without shipping a new page.

The second job is material sharing. Creating a material per widget would multiply the
material switches in a frame by the widget count. The registry caches materials keyed by
(page, pass), so every icon on the same page with the same pass yields the same material, and
the draw order — which follows the widget tree, not the material — still batches reasonably.

## State

```text
RECORD TextureInfo
  page : text     # the texture file the icon lives on
  rect : Rect     # the icon's sub-rectangle, in TEXELS of that page, not normalised

# both are process-wide and survive across screens; both are cleared on a UI reset
textures : map<text, TextureInfo>          # icon name -> where it is
materials: map<(page, pass), Material>     # the shared material cache
```

**Invariants**

- The rectangle is stored in texels of the page, as authored. Normalisation happens at draw
  time, when the page's actual resolution is known — which it is not at load time, because
  the page may not be resident yet. See
  [`UIStaticItem.cpp`](../Static/UIStaticItem.cpp.md).
- An icon name is unique across every description document. Later documents *replace* earlier
  entries, which is how a style overrides one icon.
- Clearing the registry must also clear the material cache, because the cache's keys name
  pages that the registry is about to forget.

## `ParseShTexInfo` — three forms

**Contract** — loads icon descriptions. Three entry points for three document shapes, all
writing into the same registry.

*By path and filename* — load the named document from the given directory and parse it in
the multi-page shape, with override enabled. This is the bulk loader used by the startup
scan in [`ui_base.cpp`](../ui_base.cpp.md).

*By filename alone* — load through the style-aware path lookup, and parse the **single-page**
shape: a `file_name` element naming one page, then a flat list of `texture` elements each
with an identity and a rectangle. Silently does nothing if the document is absent.

*From an already-loaded document* — parse the **multi-page** shape: a list of `file` elements,
each naming a page and containing that page's `texture` elements. Takes a flag saying whether
later definitions may override earlier ones.

```text
FUNCTION parse(doc, allow_override)
  FOR EACH "file" element f
    page = f.attribute "name"
    descend into f
    FOR EACH "texture" element t
      info = { page, rect from t's x, y, width, height }
      id   = t.attribute "id"
      IF id is not yet registered THEN register it
      ELSE IF allow_override THEN replace it
    restore the previous position in the document
```

**Invariants** — the rectangle is authored as an origin plus an extent, and stored as a pair
of corners: `x2 = x + width`. Every consumer reads corners.

**Notes** — the two document shapes exist because the older games ship the single-page form
and the newest ships the multi-page form. Both are frozen data; both must be read.

## `InitTexture` — two forms

**Contract** — resolve an icon name to a material and a rectangle, creating and caching the
material on first use. Returns whether the name was found in the registry.

```text
FUNCTION init_texture(icon, pass, out_material, out_rect) -> bool
  IF icon IS registered
    key = (its page, pass)
    IF key not in the material cache THEN create a material for (pass, page) under it
    out_material = materials[key]
    out_rect     = the registered rectangle
    RETURN true
  # not registered: treat the name as a texture file in its own right
  out_material = a material created for (pass, icon)
  RETURN false
```

**Invariants** — **the failure path is not a failure.** An unregistered name is taken to be a
plain texture path, so a layout may reference a loose texture that has no atlas entry. Callers
use the return value to distinguish "an atlas icon with a known size" from "a whole texture
whose size must be asked of the device". Every caller in the chapter relies on this.

The second form does the same and additionally pushes the result into a drawable item:
material, sub-rectangle, and the item's size set to the rectangle's extent — so an icon
defaults to its authored size.

## `FindItem` — three forms

**Contract** — look up an icon, optionally falling back to a second name.

```text
FUNCTION find_item(icon, fallback, out) -> bool
  IF icon IS registered THEN out = it; RETURN true
  IF fallback IS registered THEN out = it; RETURN true
  RETURN false
```

The return-value form asserts the lookup succeeded and names both the icon and the fallback
in the failure message. The out-parameter forms report absence instead. Both policies are
exported to scripts, and which one a caller picks *is* its error handling.

## `GetTextureRect` / `GetTextureFileName` / `GetTextureWidth` / `GetTextureHeight` / `ItemExist`

**Contract** — derived queries. The width/height pair exists in two flavours matching the two
failure policies: one asserts, one reports. `ItemExist` is the pure membership test, used by
widgets that probe for one of several possible art sets — for example the spin box, which
asks whether the newest game's art is present before falling back to the older games'
(see [`UICustomSpin.cpp`](../SpinBox/UICustomSpin.cpp.md)).

## `GetTextureShader`

**Contract** — creates a material for an icon's page with the engine's default pass, without
consulting the cache. Aborts if the icon is unknown.

**Notes** — bypassing the cache means each call creates a new material for a page that
probably already has one. It is used rarely and is a defect rather than a decision; a rebuild
routes it through the same cache as `InitTexture`.

## `FreeTexInfo` / `FreeCachedShaders`

**Contract** — drop the whole registry (and with it the material cache), or just the material
cache. Called on a UI reset, before the registry is repopulated from data.

**Invariants** — releasing the material cache releases the widget layer's references to
device resources. Every widget still holding a material keeps it alive until it too is
released, so the order of teardown between screens and this registry does not matter.

## `IsSh`

**Contract** — a private predicate: a name containing no path separator is an icon name, one
that contains a separator is a texture path. Declared, and not used by anything in this file.

**Notes** — the distinction it encodes is real and is applied implicitly by `InitTexture`'s
fallback, but this predicate itself is dead. A rebuild should either apply it explicitly at
the lookup or drop it.
