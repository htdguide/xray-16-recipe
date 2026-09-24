# src/xrUICore/SpinBox/UICustomSpin.h

> Declares the spin-box base implemented in [`UICustomSpin.cpp`](UICustomSpin.cpp.md), and the four-operation contract every spin subtype must satisfy.

**Needs** — [`UICustomSpin.cpp`](UICustomSpin.cpp.md) · [`Windows/UIWindow.h`](../Windows/UIWindow.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md)
**Used by** — [`UICustomSpin.cpp`](UICustomSpin.cpp.md) · [`UISpinNum.cpp`](UISpinNum.cpp.md) · [`UISpinNum.h`](UISpinNum.h.md) · [`UISpinText.cpp`](UISpinText.cpp.md) · [`UISpinText.h`](UISpinText.h.md) · [`UIXmlInitBase.cpp`](../XML/UIXmlInitBase.cpp.md)
**Tier floor** — T3: a declaration with an abstract contract.

## Purpose

Declares the type implemented in [`UICustomSpin.cpp`](UICustomSpin.cpp.md). The substantive
content here is the *variation point*: a spin box is a widget plus four questions about a
value it does not understand.

```text
FUNCTION can_press_up()   -> bool    # is there a next value
FUNCTION can_press_down() -> bool    # is there a previous value
FUNCTION inc_val()                   # move to the next value and redisplay
FUNCTION dec_val()                   # move to the previous value and redisplay
```

That is the whole abstraction, and it is enough for both shipped subtypes — a bounded number
([`UISpinNum`](UISpinNum.h.md)) and a list of localized labels
([`UISpinText`](UISpinText.h.md)). The two predicates are consulted every frame to grey out
an arrow at a limit; the two mutators are called in bursts by the auto-repeat without
re-checking the predicates in between, so they must be safe at the limit.

A spin box is also an options item, so it participates in the settings screen's
read / back up / save / undo protocol; that half is implemented by the subtypes, since only
they know the value's type.

## Exported units

- `CUICustomSpin` — the base.
- `InitSpin(position, size)` — build the box art and the two arrows from whichever generation
  of the shipped art is installed. The height is fixed, not taken from `size`.
- `Draw` / `Update` / `SendMessage` / `Enable`.
- `OnBtnUpClick` / `OnBtnDownClick` — the notification funnel every value change passes
  through; subtypes override, change the value, then delegate up.
- `GetText` — the currently displayed text.
- `SetTextColor` / `SetTextColorD` — enabled and disabled text colours.
- The protected four: `CanPressUp`, `CanPressDown`, `IncVal`, `DecVal`.

**Notes** — the text block is held as a plain owned object rather than a tree child, which is
why the type has a destructor at all. In a rebuild it is simply a field of the widget.
