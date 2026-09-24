# src/editors/xrSdkControls/Controls/ColorPicker

## What this module is responsible for

Editing one colour with four channel sliders and a live swatch. It is the general-purpose colour widget of the control library — four rows of slider-plus-number over red, green, blue and alpha, a square sample that shows transparency honestly, and a toggle between decimal and hexadecimal channel display.

## Where it sits and what it rests on

It rests on [`NumericSlider`](../NumericSlider/README.md) for each channel row and on [`ColorSampleBox`](../ColorSampleBox.cs.md) for the swatch. Nothing rests on it: **the weather editor's colour rows do not use this control.** They use three separate real-valued rows in a sub-grid, and the picker that would have opened this widget is disabled — see [`property_color_base.cpp`](../../../xrWeatherEditor/property_color_base.cpp.md). Connecting the two is the most valuable single change a rebuild of this editor could make.

## The load-bearing ideas

**The swatch is the value.** The control has no stored colour field: reading the value reads the swatch, and the four sliders are views that are reconciled into it. A representation that cannot diverge is one that needs no synchronization.

**Two guards break the echo.** A suppression flag while the control writes into its own children, and an equality test before storing. Either alone is insufficient; together they let a listener write the value back on every change without recursion.

**Disabling alpha forces it to full.** An alpha channel that is hidden but retains a stale value would silently multiply the colour. Hiding it sets it opaque.

## The twins

| File | Role |
|---|---|
| [`ColorPicker.cs`](ColorPicker.cs.md) | Four channel sliders and a swatch, reconciled into one colour without an echo loop |
| [`ColorPicker.Designer.cs`](ColorPicker.Designer.cs.md) | The layout: four rows, a square swatch, and the size envelope that keeps it usable when docked |

The directory also carries a resource file holding the swatch's checkerboard reference; it is build data with no decision in it and has no twin.
