# src/editors/xrSdkControls/Controls/IntegerSlider

## What this module is responsible for

Editing one whole number with both gestures at once: a track bar to drag and a spin box to type. It is the integer half of a pair whose other half is [`NumericSlider`](../NumericSlider/README.md).

## Where it sits and what it rests on

It rests on [`IntegerUpDown`](../IntegerUpDown/README.md) for the numeric half and on the host toolkit's track bar for the other. It is used directly by editor panels that want an integer with a visible range.

## The load-bearing ideas

**No stored value.** The spin box is authoritative and the track mirrors it, so the two cannot disagree by construction.

**The echo break is an equality test, not a flag.** Both children hold the *same* integer with no conversion, so "does the other one already hold this" is an exact question and answering it stops the recursion. Its fractional sibling cannot do this and needs a flag instead — the difference is the whole reason the two exist separately.

**The split between the two children is one number.** Widening the spin box narrows the track by the same amount, so the control always exactly fills its own width, and neither child can be squeezed below its own minimum.

## The twins

| File | Role |
|---|---|
| [`IntegerSlider.cs`](IntegerSlider.cs.md) | The pairing: forwarded properties, the echo break, and the movable boundary between the two children |
| [`IntegerSlider.Designer.cs`](IntegerSlider.Designer.cs.md) | The layout: track left, spin box right, one row tall, with a declared width envelope |

A resource file accompanies the layout; it holds no decision and has no twin.
