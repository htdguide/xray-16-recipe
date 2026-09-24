# src/editors/xrSdkControls/Controls

## What this module is responsible for

Every widget the editor is built from that the host toolkit does not already provide: a property grid that understands drag and double-click gestures, a tree addressed by path with filter and search attached, a colour picker, and four numeric input forms.

None of it knows anything about weather, levels or the engine. The five [interfaces](Interfaces/README.md) are the whole vocabulary between these widgets and whatever supplies their data, which is what lets the same controls serve a weather editor and, in principle, any other tool.

## Where it sits and what it rests on

It rests on the host widget toolkit, on one third-party property-bag library that supplies the grid's row-description type, and on nothing else in the repository. It is chapter 29's leaf: [`xrWeatherEditor`](../../xrWeatherEditor/README.md) builds its panels from these controls, and nothing else in the engine references them.

## The load-bearing ideas

Six ideas run through the directory and the individual twins are terse because they are named here.

**A property is a binding, not a value.** The grid holds no copies. Reading a row calls a getter; editing it calls a setter; there is no apply step and no synchronization. Stated in [`IProperty`](Interfaces/IProperty.cs.md), and it is the same decision the engine states on its side of the boundary.

**Optional capabilities, tested not assumed.** A property may claim to be draggable or double-clickable. The grid asks; a property that does not claim a capability simply does not get that gesture. Adding a gesture never breaks an existing property.

**Gestures speak pixels; properties speak their own units.** The widget knows that the mouse moved forty pixels. Only the property knows what forty pixels of fog density is. That split is what stops a 0-to-1 field and a 0-to-100000 field feeling identical under the same drag.

**Pull, never push.** Trees are cleared and refilled by their source; grids read on paint. There is no incremental reconciliation path anywhere in this library, so no view can drift from its data. The cost is losing expansion and selection state on a refresh, and the editor accepts it.

**Paired widgets must break their own echo.** Three controls here pair two views of one value — slider and spinner, four sliders and a swatch. Each needs a guard against writing into its own children and hearing it back. The two solutions used are an exact equality test (sound only when both views hold the identical value) and a suppression flag (needed when one view is a lossy projection of the other), and the choice between them is the difference between [`IntegerSlider`](IntegerSlider/README.md) and [`NumericSlider`](NumericSlider/README.md).

**Parsing is forgiving; storing is strict.** A half-typed number is an incomplete edit, not an error, and the previous value stands. A committed value outside a declared range is a caller error.

## The twins

| File | Role |
|---|---|
| [`PropertyGrid.cs`](PropertyGrid.cs.md) | The grid: middle-drag scrubbing, double-click routing, and a splitter position that survives a restart |
| [`ColorSampleBox.cs`](ColorSampleBox.cs.md) | A swatch that shows transparency honestly, by compositing over a checkerboard it owns |
| [`MenuButton.cs`](MenuButton.cs.md) | A button that drops a menu instead of firing an action |
| [`Interfaces/`](Interfaces/README.md) | The five contracts: the binding, the join, two optional gestures, and a tree's data source |
| [`ColorPicker/`](ColorPicker/README.md) | Four channel sliders and a live swatch — complete, and not used by the weather editor |
| [`IntegerUpDown/`](IntegerUpDown/README.md) | A spin box over whole numbers, with a held-button acceleration schedule |
| [`IntegerSlider/`](IntegerSlider/README.md) | Track bar plus spin box over a whole number |
| [`NumericSlider/`](NumericSlider/README.md) | Track bar plus spin box over a fractional number, with the track as a lossy projection |
| [`NumericSpinner/`](NumericSpinner/README.md) | A fractional spin box with an unbounded drag gesture |
| [`TreeNode/`](TreeNode/README.md) | One tree entry: kind, path key, open and closed icons, own selection flag |
| [`TreeView/`](TreeView/README.md) | The tree: addressed by path, painting its own multiple selection |
| [`TreeViewFilterPanel/`](TreeViewFilterPanel/README.md) | Narrow a tree by detaching what does not match — complete, and never wired up |
| [`TreeViewSearchPanel/`](TreeViewSearchPanel/README.md) | Find in a tree by tinting matches and stepping through them |
