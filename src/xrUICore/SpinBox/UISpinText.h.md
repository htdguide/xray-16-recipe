# src/xrUICore/SpinBox/UISpinText.h

> Declares the named-choice spin box implemented in [`UISpinText.cpp`](UISpinText.cpp.md).

**Needs** — [`UISpinText.cpp`](UISpinText.cpp.md) · [`UICustomSpin.h`](UICustomSpin.h.md)
**Used by** — [`UIMapList.cpp`](../../xrGame/ui/UIMapList.cpp.md) · [`UISpinText.cpp`](UISpinText.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3: a declaration.

## Purpose

Declares the type implemented in [`UISpinText.cpp`](UISpinText.cpp.md). One decision is
visible only here: the choice record keeps the original name, the translated label and the
token identifier side by side, so the widget can display one spelling and store another
without a second lookup at save time.

The other is a deliberate omission — the base's two step mutators are declared empty, which
turns the accelerating auto-repeat off for this subtype. A text spin box moves one choice per
click and never sweeps.

## Exported units

- `CUISpinText` — the spin box over a list of named choices.
- `AddItem_(name, id)` — append a choice, translating its label on insert.
- `GetTokenText` — the current choice's *untranslated* name, which is what the rest of the
  engine compares and stores.
- The options-item protocol: `SetCurrentOptValue` (which also populates the list from the
  setting's token list), `SaveBackUpOptValue`, `SaveOptValue`, `UndoOptValue`,
  `IsChangedOptValue`.
- `OnBtnUpClick` / `OnBtnDownClick` — the only ways the cursor moves.
- `CanPressUp` / `CanPressDown` — whether a next or previous choice exists.
