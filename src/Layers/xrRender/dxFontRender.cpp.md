# src/Layers/xrRender/dxFontRender.cpp

> The renderer's filling of the font port: turns a frame's worth of queued text lines into quads, expanding key-binding placeholders as it goes, and draws them in as few batches as the scratch buffer allows.

**Needs** — [`dxFontRender.h`](dxFontRender.h.md) · [`Include/xrRender/FontRender.h`](../../Include/xrRender/FontRender.h.md) · [`xrEngine/GameFont.h`](../../xrEngine/GameFont.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxFontRender.h`](dxFontRender.h.md)
**Tier floor** — T1: it writes a fixed vertex layout into a mapped scratch buffer with an explicit stride.

## Purpose

The font object owns the metrics — which texel rectangle each character occupies in the atlas, how wide each glyph is, how lines are spaced — and accumulates a list of strings to draw. This file owns the other half: converting that list into geometry.

The split matters because the font object is used by the game and by the tools, and neither should know what a vertex is; and because the *only* renderer-specific decision in text drawing is how many quads fit in one buffer lock.

## State

```text
RECORD FontRenderer
  material : Material    # created from a named shader and the atlas texture
  geometry : VertexFormat  # position+colour+one texture coordinate, drawn as a quad list
```

Stateless between frames: everything else comes from the font object passed in.

## `initialize(shader_name, texture_name)`

**Contract** — creates the material from the two names and the vertex format bound to the renderer's scratch vertex pool and its **shared quad index buffer**. Allocates device resources.

**Invariants** — the quad index buffer is a renderer-wide constant: the repeating pattern that turns every four consecutive vertices into two triangles. Text never supplies its own indices, which is why the vertex order below is fixed.

## `render(font)`

**Contract** — draws every string the font has queued, then leaves the queue for the font to clear. Locks the scratch vertex pool once per batch and issues one draw per batch. Asserts that a frame is in progress. Allocates nothing on the heap except a stack buffer per string.

```text
FUNCTION render(font)
  bind material

  IF font's atlas dimensions are not yet known THEN
    ask the draw stream which texture is bound to the base sampler
    font.atlas_size = that texture's width and height
    font.glyph_height_in_texture_coords = font.pixel_height / atlas_size.height
    mark the font valid

  i = 0
  WHILE i < font.line_count
    # 1. First-fit batching
    count = 1 ; total = expanded_length(font.line[i])
    WHILE i + count < font.line_count
      next = expanded_length(font.line[i + count])
      IF total + next >= max_characters_per_batch THEN BREAK
      total = total + next ; count = count + 1

    # 2. Fill
    buffer = lock_vertex_pool(total * 4 vertices)
    FOR each of the next `count` lines
      emit_line(line, buffer)
    written = buffer.position - start

    # 3. Draw
    unlock_vertex_pool(written)
    IF written > 0 THEN draw_triangles(written vertices, written / 2 triangles)
    i = i + count
```

**Invariants**

- The batch is sized by a **character count**, not a vertex count, and the count is the *expanded* length: a line containing a key-binding placeholder expands to however many characters the current binding's name has. The expansion must be measured the same way in the sizing pass and the fill pass or the lock overflows. The source measures it as `characters + binding_text_length − 2 × binding_count`, the subtraction removing the two-character placeholder itself.
- A single line longer than the batch limit is emitted alone and overflows; the limit is chosen large enough that no shipped string reaches it.
- The atlas dimensions are discovered at first draw by *asking the draw stream what texture is currently bound* to the base sampler, rather than by asking the texture the material names. This is how a font whose atlas is chosen by the shipped material gets its metrics without the font object knowing the material. It requires the material to be bound first, which is why the bind precedes the check.

### `emit_line` — laying out one string

```text
FUNCTION emit_line(line, buffer)
  text = line.text decoded to wide characters if the font is multi-byte
  IF text is empty THEN RETURN

  x = floor(line.x) ; y = floor(line.y)
  y2 = y + line.height * vertical_ui_scale

  IF line.alignment is not left THEN
    width = font.measure(text)
    IF centred THEN x = x - floor(width * 0.5) * horizontal_ui_scale
    IF right    THEN x = x - floor(width)

  top_colour = line.colour
  bottom_colour = line.colour
  IF font wants a gradient THEN bottom_colour = line.colour with RGB halved, alpha kept

  # half-texel correction, inverted: the vertex program applies the older API's
  # half-pixel offset, so the position is pre-shifted back by half a pixel
  x = x - 0.5 ; y = y - 0.5 ; y2 = y2 - 0.5

  FOR EACH character IN text
    IF character is the key-binding marker THEN
      take the NEXT character as an action identifier
      FOR EACH character IN the text of that action's current binding
        imprint(character)
    ELSE
      imprint(character)
```

**Invariants**

- Positions are floored to whole pixels before anything else. Text that lands on a half-pixel is blurred by the atlas's bilinear filter, and the engine's UI is authored assuming crisp glyphs.
- The gradient halves red, green and blue and **keeps alpha**. Halving alpha would fade the bottom of every glyph.
- The key-binding marker is a single reserved character followed by one character holding an action identifier. The identifier is a single byte, which caps the action set at 255 — the source asserts this at compile time. A rebuild that grows the action set past 255 must widen the encoding, and the shipped string tables contain these markers, so the marker character itself is frozen.
- The binding's text is looked up *at draw time*, every frame. That is what makes a rebound key change every on-screen hint immediately, and it is why the batch sizing must re-measure rather than cache.

### `imprint` — one glyph

```text
FUNCTION imprint(glyph, ...)
  (u, v, advance) = font.texture_coords_of(glyph)
  IF advance is not zero THEN
    emit four vertices, in this order:
      (x,           y2, bottom_colour, u,          v + glyph_height)
      (x,           y,  top_colour,    u,          v)
      (x + advance, y2, bottom_colour, u + width,  v + glyph_height)
      (x + advance, y,  top_colour,    u + width,  v)
  x = x + advance * font.horizontal_interval
  IF the font is multi-byte THEN
    x = x - 2
    IF the glyph is one that needs a following space THEN x = x + font.space_advance
```

**Invariants**

- The four-vertex order is bottom-left, top-left, bottom-right, top-right. It is fixed by the shared quad index buffer; any other order draws a bow tie.
- A glyph with zero advance emits **no vertices but still advances nothing** — it is a character the atlas has no cell for. The batch sizing counts it, so the buffer is over-locked rather than under-locked, which is the safe direction.
- The `−2` and the conditional space in the multi-byte path are the East Asian text adjustment: those atlases are laid out with two pixels of built-in padding per cell that must be removed, and characters that do not imply their own spacing get one space's worth added back. Both are authored constants that match the shipped atlases.

**Notes** — The commented-out half-texel offsets in the texture coordinates are a deliberate removal, not dead code: the correction was moved from the texture coordinate to the position (the `−0.5` above), and applying it in both places shifts every glyph by a full texel.
