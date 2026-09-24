# src/xrUICore/SpinBox/UISpinNum.h

> Declares the two numeric spin boxes implemented in [`UISpinNum.cpp`](UISpinNum.cpp.md).

**Needs** — [`UISpinNum.cpp`](UISpinNum.cpp.md) · [`UICustomSpin.h`](UICustomSpin.h.md)
**Used by** — [`UIKickPlayer.cpp`](../../xrGame/ui/UIKickPlayer.cpp.md) · [`UIMPServerAdm.cpp`](../../xrGame/ui/UIMPServerAdm.cpp.md) · [`UISpinNum.cpp`](UISpinNum.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the two types implemented in [`UISpinNum.cpp`](UISpinNum.cpp.md): a spin box over a
bounded integer and one over a bounded real. They are separate types rather than one
parameterized type because they bind to differently typed settings and, as the implementation
records, they behave differently at their bounds.

The declaration fixes the defaults a layout inherits when it names no range: zero to one
hundred, stepping by one for the integer type and by a tenth for the real one.

## Exported units

- `CUISpinNum` — the integer spin box. `SetMin` / `SetMax` / `Value` / `InitSpin`.
- `CUISpinFlt` — the real spin box. `SetMin` / `SetMax` / `InitSpin`. It exposes no value
  reader, which is an omission rather than a decision — its value reaches the world only
  through the setting it saves to.
- Both implement the options-item protocol (`SetCurrentOptValue`, `SaveBackUpOptValue`,
  `SaveOptValue`, `UndoOptValue`, `IsChangedOptValue`) and the base spin box's four-operation
  contract, plus `OnBtnUpClick` / `OnBtnDownClick`.
