# src/editors/xrSdkControls/Properties

## What this module is responsible for

The control library's module identity and its one embedded image. Two generated files and a resource table; neither contains an algorithm.

## Where it sits and what it rests on

It rests on nothing. [`ColorSampleBox`](../Controls/ColorSampleBox.cs.md) is the only consumer of the image.

## The load-bearing idea

**The checkerboard ships inside the module, not beside it.** The control library is loaded from a directory the editor host chooses, and a missing sibling file would make a colour swatch silently wrong rather than loudly absent. A rebuild embeds the image, generates it in code (it is a two-colour checkerboard), or draws it directly — any of the three. What it must not do is load it through the game's virtual filesystem, which this library deliberately knows nothing about.

## The twins

| File | Role |
|---|---|
| [`AssemblyInfo.cs`](AssemblyInfo.cs.md) | The module's identity: name, vendor, a version nothing reads, and a declaration that it is not for foreign callers |
| [`Resources.Designer.cs`](Resources.Designer.cs.md) | Typed access to the one embedded image, the transparency checkerboard |

The resource table itself is build data and has no twin.
