# src/editors/xrSdkControls/Controls/IntegerUpDown

## What this module is responsible for

A spin box over whole numbers: typed entry with a parse that tolerates half-finished input, optional hexadecimal display, a range that is never violated, and a held-button acceleration schedule.

It exists because the host toolkit's own spin box works in decimal fractions, and the editor needs integers — colour bytes, flag masks, counts — shown sometimes in base sixteen.

## Where it sits and what it rests on

It rests on the host toolkit's up-down base and on nothing in this library. [`IntegerSlider`](../IntegerSlider/README.md) embeds it.

## The load-bearing ideas

**The text and the value are two representations, reconciled at four named moments** — a spin step, focus loss, a programmatic read, and the end of an initialization bracket — and at no other time. Two flags say which side is stale.

**A failed parse is an incomplete edit, not an error.** A lone minus sign is how a negative number begins; rejecting it would make negative values untypeable. The previous value stands and the author keeps typing. Every numeric input in every language meets this, and this is the answer.

**The range is never inverted and never violated.** Setting a minimum above the maximum drags the maximum with it. A value written outside the range is rejected as a caller error rather than clamped, because the caller declared the range — except inside an initialization bracket, where order is unknown and the value is clamped on close.

**Acceleration resets at saturation.** Holding the button until the value reaches an end and then reversing must not produce one enormous step.

## The twins

| File | Role |
|---|---|
| [`IntegerUpDown.cs`](IntegerUpDown.cs.md) | The spin box: the text/value split, key filtering, the range rules, and the acceleration state machine |
| [`IntegerUpDownAccelerationCollection.cs`](IntegerUpDownAccelerationCollection.cs.md) | The acceleration schedule, kept sorted by hold time so it can only be walked forward |
| [`IntegerUpDownAcceleration.cs`](IntegerUpDownAcceleration.cs.md) | One schedule entry: after this long, step by this much; both non-negative |
