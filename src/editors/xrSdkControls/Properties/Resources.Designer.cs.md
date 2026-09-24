# src/editors/xrSdkControls/Properties/Resources.Designer.cs

> Typed access to the one image compiled into the control library: the transparency checkerboard.

**Needs** — [`ColorSampleBox.cs`](../Controls/ColorSampleBox.cs.md)
**Used by** — [`ColorSampleBox.cs`](../Controls/ColorSampleBox.cs.md)
**Tier floor** — T4: generated lookup over an embedded resource table.

## Purpose

Generated accessors over the module's embedded resource table. The table holds exactly one entry — the checkerboard tile [`ColorSampleBox`](../Controls/ColorSampleBox.cs.md) paints behind a colour sample to make its transparency visible.

## State

```text
RECORD Resources
  background : image     # a small two-tone checkerboard, tiled
  culture    : optional<locale>   # forced locale for lookups; unset in practice
```

## Notes

The whole file is one decision: **the checkerboard ships inside the module rather than beside it as a file**. That matters because the control library is loaded from a directory the editor host chooses, and a missing sibling file would make a colour swatch silently wrong rather than loudly absent. A rebuild embeds it, generates it in code (it is a two-colour checkerboard), or draws it directly — any of the three; what must not happen is loading it from the game's virtual filesystem, which the editor's control layer deliberately knows nothing about.

The locale override exists in every generated table of this kind and is unused here; there are no localized strings.
