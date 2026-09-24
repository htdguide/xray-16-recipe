# src/xrUICore/TrackBar/UITrackBar.h

> Declares the options slider implemented in [`UITrackBar.cpp`](UITrackBar.cpp.md).

**Needs** — [`UITrackBar.cpp`](UITrackBar.cpp.md) · [`InteractiveBackground/UI_IB_Static.h`](../InteractiveBackground/UI_IB_Static.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md)
**Used by** — [`UIMPPlayersAdm.cpp`](../../xrGame/ui/UIMPPlayersAdm.cpp.md) · [`UITrackBar.cpp`](UITrackBar.cpp.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UITrackBar.cpp`](UITrackBar.cpp.md). Three decisions are
visible only here.

**One widget serves two value types.** A track bar binds either to an integer setting or to a
real one, chosen by a flag, and the five numeric fields — value, bounds, step and backup —
are one storage slot reinterpreted accordingly. In C++ that is a union and the flag is the
only thing keeping it honest; in a rebuild it is a tagged number or two widget types, but the
flag must survive in some form because both kinds of setting exist and both are edited by this
control.

**An integer track bar doubles as a checkbox.** The declaration exposes a boolean face over
the integer value, valid only for that type. The options screens use a zero-to-one integer
bar wherever a switch is wanted, which is why the toolkit has no separate settings toggle.

**The numeric readout is a public field, not an encapsulated one.** The label widget and its
format string are exposed so a layout can position and style the readout independently of the
bar. The readout is drawn only while the label is enabled, which is how a layout opts in.

## Exported units

- `CUITrackBar` — the slider.
- `InitTrackBar(position, size)` — build the bar and knob art, probing two asset generations.
- `m_static` / `m_static_format` — the optional numeric readout and its format template,
  which is resolved through the string table so it can be localized.
- `SetInvert` / `GetInvert` — make the maximum the left end.
- `SetType(is_real)` — select which half of the value representation is live.
- `SetStep` / `SetOptIBounds` / `SetOptFBounds` — the detent size and the range; setting a
  range re-clamps the current value.
- `SetBoundReady` — declare the range already correct, so reading the setting does not
  overwrite it. For settings whose range is computed from the hardware.
- `GetIValue` / `GetFValue` — read the value in either representation.
- `GetCheck` / `SetCheck` — the toggle face, integer bars only.
- `StepLeft` / `StepRight` — move one detent in a *screen* direction; inversion is applied
  inside, so callers never reason about it.
- `OnMouseAction` / `OnKeyboardAction` / `OnControllerAction` — drag, wheel, bound arrow keys
  and a horizontally deflected stick.
- `OnMessage("set_default_value")` — the options screen's reset, which centres the value in
  its range.
- The options-item protocol: `SetCurrentOptValue`, `SaveBackUpOptValue`, `SaveOptValue`,
  `UndoOptValue`, `IsChangedOptValue`.
- `Draw` / `Update` / `Enable`.
