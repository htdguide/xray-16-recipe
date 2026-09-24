# src/Layers/xrRender/ColorMapManager.cpp

> Points the two colour-grading lookup slots at named textures, and never lets a grading texture leave memory once loaded.

**Needs** — [`ColorMapManager.h`](ColorMapManager.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`SH_Texture.h`](SH_Texture.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it rebinds the underlying device surface of one texture object to another's, which is a handle-level operation below the texture abstraction.

## Purpose

The weather system grades the finished frame through a pair of lookup textures and cross-fades between them as the time of day advances. Shaders do not know which grading texture is current; they sample two fixed *named* slots. This file is the indirection that makes that work: it keeps two permanently-existing texture objects under reserved names and swaps the device surface *inside* them as the weather changes.

It is a separate file because of the second half of its job, which is a lifetime decision: a grading texture, once loaded, is never released. The weather cycle returns to the same few textures every in-game day, and letting the ordinary texture cache evict them would cause a load stall at dawn.

## State

```text
RECORD ColorMapManager
  slots      : list<texture> of length 2      # named "$user$cmap0" and "$user$cmap1"
  slot_names : list<text>    of length 2      # what each slot currently points at
  cache      : map<text, texture>             # every grading texture ever bound, kept alive
```

**Invariants**

- The two slot textures are created once, at construction, under names in the engine's reserved user namespace, and they exist for the renderer's whole life. Their identity never changes; only the device surface behind them does. This is what lets shipped shaders name the slots statically.
- `slot_names[i]` is the authoritative record of what slot `i` holds. A request naming the same texture again does nothing at all — this is the entire reason the record exists, since the weather system re-asserts its current pair every frame.
- The cache is never pruned. It is bounded by the number of distinct grading textures in the shipped weather configuration, which is a couple of dozen small images.
- An empty name clears the slot to no texture. Shaders must tolerate sampling an unbound grading slot, which in practice means the pass is skipped when grading is off.

## `set_textures(name0, name1)`

**Contract** — points the two slots at the two named textures. Called every frame by the weather blend. Returns immediately when nothing changed. May block on a texture load the first time a name is seen, which is why the weather system asserts its pair early in the frame rather than at the point of use.

```text
FUNCTION update_slot(name, index)
  IF name = slot_names[index]    RETURN        # the common case, every frame

  slot_names[index] = name

  IF name is empty
    slots[index].bind_surface(none)
    RETURN

  texture = cache[name]  OR  (load texture by name, then insert into cache)
  slots[index].bind_surface(texture.surface)
```

**Notes** — Rebinding the surface is the load-bearing trick and the only thing in this file a rebuild must think about. The slot object is a *stable handle* whose contents are reassigned; the shipped shader names the handle. On a graphics API with immutable texture views, the equivalent is a descriptor slot the renderer rewrites, and the grading texture cache stays exactly as it is.

The surface handle is reference-counted by the device on the backend the original was written against, so the code takes a reference, hands it over, and drops its own — a detail that disappears entirely under any other ownership model. What must survive is that the cache, not the slot, owns the texture.
