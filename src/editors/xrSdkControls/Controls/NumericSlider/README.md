# src/editors/xrSdkControls/Controls/NumericSlider

## What this module is responsible for

Editing one fractional number with a track bar and a spin box together. It is the form almost every real-valued weather field would use, and the one [`ColorPicker`](../ColorPicker/README.md) builds its four channel rows from.

## Where it sits and what it rests on

It rests on the host toolkit's own track bar and fractional spin box. [`ColorPicker`](../ColorPicker/README.md) is its main consumer.

## The load-bearing ideas

**The track holds a projection, not the value.** A track bar has integer positions and a fog density does not, so the control defines a resolution — a fixed number of track positions spanning the whole range — and maps the value onto it. The map is lossy in one direction.

**The resolution belongs to the control, not to the value.** A fixed pixel-to-value mapping would make a 0-to-1 property and a 0-to-1000 property feel different under the same drag. A fixed *position count* makes every range drag identically, which is what an author wants.

**Dragging quantizes; typing does not.** A drag snaps the value to the projection's resolution; the spin box writes an exact value. That asymmetry is correct — a drag is a coarse gesture — and it is why the echo break here must be a suppression flag rather than the equality test its integer sibling uses.

## The twins

| File | Role |
|---|---|
| [`NumericSlider.cs`](NumericSlider.cs.md) | The projection, the echo break, and the movable boundary between the two children |
| [`NumericSlider.Designer.cs`](NumericSlider.Designer.cs.md) | The layout, identical to the integer variant but over the stock fractional spin box |

A resource file accompanies the layout; it holds no decision and has no twin.
