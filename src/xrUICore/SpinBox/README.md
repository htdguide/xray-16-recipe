# src/xrUICore/SpinBox — steppers

> A framed value with an up button and a down button. Three of them: integer, real and
> token-list. All three are settings controls.

Part of [chapter 15](../README.md).

## What this directory is responsible for

The common base owns the assembly — a three-segment frame, two small buttons, and a text
control showing the current value — and the repeat behaviour that makes a held button keep
stepping. It leaves four questions abstract: may the value go up, may it go down, step it up,
step it down. Each of the three concrete spinners answers those four and nothing else.

All three also implement the settings-control protocol, so a spinner in an options screen
reads its console variable on open, backs it up, commits on accept and restores on cancel.

## The load-bearing ideas

**The base is a template method with exactly four holes.** That is the whole design: the frame,
the buttons, the text, the press-and-hold repeat and the enabled propagation are written once;
the value model is written three times. A rebuild gets the same shape from any mechanism.

**Press-and-hold repeats after an initial delay, then faster.** Two separate delays —
a pause before the first repeat and the interval between repeats afterwards — because a single
interval either feels unresponsive or overshoots. The values are tuning, not derivation.

**The bounds belong to the value, not to the widget.** Minimum, maximum and step live on the
concrete spinner, so the integer and real variants carry them in their own types rather than
sharing a real-valued pair. That matters because the integer spinner must not round-trip
through a real.

**The token spinner is a list, not a range.** It holds (original name, localized name,
identifier) triples, steps through them by index, and stores the identifier. Its increment and
decrement do nothing in the value sense — the whole change is the index move — which is why its
two step operations are empty and the item setter does the work.

## The twins

| Twin | Role |
|---|---|
| [`UICustomSpin.cpp`](UICustomSpin.cpp.md) | The assembly and the press-and-hold repeat, with four questions left to the concrete spinner |
| [`UICustomSpin.h`](UICustomSpin.h.md) | The base's declaration and the four abstract operations |
| [`UISpinNum.cpp`](UISpinNum.cpp.md) · [`UISpinNum.h`](UISpinNum.h.md) | The integer and real spinners: bounds, step, and the settings protocol for each |
| [`UISpinText.cpp`](UISpinText.cpp.md) · [`UISpinText.h`](UISpinText.h.md) | The token-list spinner: (name, localized name, identifier) triples stepped by index |
