# src/editors/xrWeatherEditor/property_converter_float.cpp

> Every real number in the editor is shown to three decimal places — and that decision is frozen into the file the engine reads.

**Needs** — [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Used by** — [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Tier floor** — T1: the precision it prints at is the precision the authored configuration file carries.

## Purpose

The text rendering of a real-valued grid row, in both directions.

It looks trivial and it is not. **This editor writes the weather configuration files that the engine reads**, and it writes them from the same authored text the author sees. So the formatting chosen here is not a display preference; it is the precision of the shipped data.

## State

`Stateless.`

## Rendering

**Contract** — a real is rendered as a fixed-point decimal with exactly three fractional digits. Always three: no trimming of trailing zeros, no switch to exponential form for very large or very small magnitudes.

**Invariants** — the format is fixed and locale-independent on the way out.

**Notes** — three digits is the decision, and it is the right order of magnitude for what these values are: unit-interval densities and intensities, blend factors, angles in turns. A fourth digit would be noise in a value an artist is tuning by eye.

It also has consequences a rebuild must accept deliberately:

**Values are quantized at three decimals on the round trip.** A value the engine computed — an interpolated keyframe, a value loaded from a file with more precision — is shown rounded, and if the author touches that row at all, the rounded value is what is stored. Repeated edit cycles do not drift further, because three decimals is a fixed point of the round trip, but the first cycle loses precision.

**Large magnitudes render unreadably.** A view distance of several hundred metres prints as a long fixed-point string rather than switching to exponential form. Nothing in the weather model is large enough for this to matter, which is why it was never addressed.

**Very small magnitudes render as zero** — anything below half a thousandth. For a fog density in the unit interval that is the correct rounding; for a value whose useful range is finer, it would silently destroy data.

## Parsing

**Contract** — parses the author's text back to a real **using the machine's current locale**, and reports a failure as a rejected edit with the offending text named.

**Notes** — the asymmetry is a real bug worth naming: **rendering is locale-independent and parsing is locale-dependent.** On a machine whose locale uses a comma as the decimal separator, the editor prints `0.500` and then refuses to parse it back. Every real-valued row in the weather grid is affected.

A rebuild parses with the same locale-independent rules it prints with. This is the single most consequential defect in this directory, because it makes the tool unusable on a large fraction of machines rather than merely imperfect on all of them.
