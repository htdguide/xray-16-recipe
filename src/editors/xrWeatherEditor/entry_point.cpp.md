# src/editors/xrWeatherEditor/entry_point.cpp

> The two exported symbols of the editor library, and the idle pump that runs a fixed-rate engine inside an event-driven application.

**Needs** — [`Include/editor/interfaces.hpp`](../../Include/editor/interfaces.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [`ide_impl.hpp`](ide_impl.hpp.md) · [`xrWeatherEngine/engine_impl.hpp`](../xrWeatherEngine/engine_impl.hpp.md) · [`window_ide.h`](window_ide.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it exports two symbols by name for a dynamic load, intercepts raw platform window messages, and initialises the platform's component runtime before any window exists.

## Purpose

This is where the editor library begins. It answers the two questions the host asks at load time — construct the editor, destroy it — and it installs the mechanism that lets a fixed-rate engine and an event-driven user interface share one thread.

That mechanism is the whole point of the file, and it is the answer to the problem stated in [`engine.hpp`](../../Include/editor/engine.hpp.md): the editor owns the message loop, the engine owns a frame, and something has to call the frame often enough that the rendered view looks live.

## State

```text
RECORD EditorLibrary            # process-global, exactly one
  editor : optional<Ide>        # invariant: set between initialize and finalize, else absent
```

**Invariants** — exactly one editor exists per process; constructing a second is a programming error and is asserted. The host's engine facade must already exist when `initialize` runs, because the editor takes a reference to it during construction.

## `initialize`

**Contract** — brings up the platform's component runtime in single-threaded-apartment mode, constructs the editor root and its main window, writes the root into the caller's slot, and returns. Does not enter the loop; the host calls `run` separately.

```text
FUNCTION initialize(out slot : Ide)
  start component runtime in single-threaded apartment mode
  FAIL WITH "already initialized" IF an editor exists
  editor = new Ide(reference to the host's engine facade)
  slot = editor
  editor.window = new MainWindow(slot, the host's engine facade)
```

**Notes** — the apartment mode is not incidental. The editor's file pickers and colour dialogs are built on a component model that requires a single-threaded apartment, and the mode must be chosen before the first window exists. A rebuild on a toolkit without that requirement drops the call; a rebuild on one *with* it must make the same choice at the same moment, because it cannot be changed afterwards.

The out-parameter is described in [`interfaces.hpp`](../../Include/editor/interfaces.hpp.md): the library allocates, the library frees, the host holds the only reference.

## `finalize`

**Contract** — destroys whatever is in the caller's slot, clears the slot, and clears the library's own record of it. Safe to call once; calling it twice destroys an already-released object.

## The main window's message interception

**Contract** — every message the main window receives is offered to the engine first. If the engine reports that it consumed the message, the window framework never sees it; otherwise the framework handles it normally.

**Notes** — this is `on_message` from [`engine.hpp`](../../Include/editor/engine.hpp.md) wired to a real window. The decision it encodes is **the engine gets first refusal on every input event**, which is what lets the three-dimensional view respond to mouse-look and keyboard movement while the surrounding application is an ordinary retained-mode interface.

The message is passed through as the platform's own four-value quadruple. That is entirely incidental — a rebuild passes whatever its windowing layer produces — but the *first-refusal ordering* is not.

## The idle pump

**Contract** — while the application has no pending messages, the idle handler runs the engine and the editor's per-frame work in a tight loop. It exits the loop when a message arrives, when the engine reports that a quit was requested, or when the engine is gone.

```text
ON application_idle
  editor.begin_idle()                      # asserts we are not already inside one
  REPEAT
    engine.advance_one_frame()             # renders the view; advances weather time
    editor.advance_one_frame()             # repaints the timeline and the view chrome
  WHILE engine exists
    AND NOT engine.quit_requested()
    AND no window message is pending
  editor.end_idle()
```

**Invariants** — the loop always runs its body at least once, so a single idle notification always produces at least one frame even under constant input. Entering and leaving the idle bracket is asserted non-re-entrant, because the editor uses "am I inside idle" to decide whether a repaint request should be served now or deferred.

**Notes** — this is the classic shape and it has the classic consequence: **a modal dialog starves the engine**, because the dialog runs its own loop and the application's idle notification never arrives. Opening a colour picker or a file browser freezes the rendered view. The editor lives with it.

The loop polls for pending messages rather than waiting on them, which means the thread never sleeps while the editor is focused — the process spins at whatever rate a frame takes. That is acceptable for a tool and would not be for a game.

A rebuild that inverts control — engine owns the loop, editor panels are drawn by the engine's own overlay — deletes this file's idle handler, its message interception, and the starvation problem with them. The recipe recommends that inversion; what must survive it is the pair of operations the engine exposes, one frame and one input offer.
