# src/xrGame/UIDialogHolder.h

> Declares the screen stack and input router implemented in [`UIDialogHolder.cpp`](UIDialogHolder.cpp.md), and the two small records it keeps its collections in.

**Needs** — [`xrEngine/pure.h`](../xrEngine/pure.h.md) · [`xrUICore/ui_debug.h`](../xrUICore/ui_debug.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`MainMenu.cpp`](MainMenu.cpp.md) · [`MainMenu.h`](MainMenu.h.md) · [`UIDialogHolder.cpp`](UIDialogHolder.cpp.md) · [`UIGameCustom.cpp`](UIGameCustom.cpp.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`UIDialogWnd.cpp`](ui/UIDialogWnd.cpp.md) · [`UIDialogWnd.h`](ui/UIDialogWnd.h.md) · [`UIMessageBoxEx.cpp`](ui/UIMessageBoxEx.cpp.md) · [`UIMpTradeWnd.cpp`](ui/UIMpTradeWnd.cpp.md)
**Tier floor** — T3: a declaration and two records

## Purpose

Declares `CDialogHolder`, the base that anything capable of showing screens inherits.
Substance is in [`UIDialogHolder.cpp`](UIDialogHolder.cpp.md).

The two records it defines:

```text
RECORD DrawEntry            # one window in the draw list
  window  : Window
  enabled : bool            # ordering: enabled entries sort before disabled ones,
                            # which is how removals are compacted at end of frame

RECORD StackEntry           # one window in the modal input stack
  window : DialogWindow
  flags  : { crosshair_was_shown, indicators_were_shown }
```

Exported units:

- `CDialogHolder` — the holder; it participates in the engine's per-frame callback list and
  in the developer overlay.
- `StartDialog`, `StopDialog`, `StartStopMenu` — open, close and toggle a screen.
- `AddDialogToRender`, `RemoveDialogToRender` — draw-list membership, independent of the
  input stack.
- `SetMainInputReceiver`, `TopInputReceiver` — the input stack's mutator and its top.
- `OnFrame` — update every window and apply deferred mutations.
- `UseIndicators`, `IgnorePause` — two questions a subclass answers about itself: whether
  it manages the heads-up display at all, and whether its screens run while the game is
  paused.
- `OnExternalHideIndicators` — surrender ownership of the heads-up display.
- `MarkForemost` — only one holder at a time owns the pointer and the focus.
- The input entry points: `IR_UIOnMouseMove`, `IR_UIOnMouseWheel`, `IR_UIOnKeyboardPress`,
  `IR_UIOnKeyboardRelease`, `IR_UIOnKeyboardHold`, `IR_UIOnTextInput`, and the three
  controller forms. Each returns whether the interface absorbed the event.
- `CleanInternals`, `DoRenderDialogs`, `UpdateCursorVisibility` — teardown, the draw pass,
  and the pointer's show/auto-hide rule.
- `GetDebugType`, `FillDebugTree`, `FillDebugInfo` — the developer overlay.
- `script_register` — declares the holder to Lua.
