# src/xrGame/ui/UIGameTutorial.h

> Declares the sequencer, the step interface every step kind must satisfy, and the two shipped step kinds.

**Needs** — [`UIGameTutorial.cpp`](UIGameTutorial.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md) · [`UIGameTutorialVideoItem.cpp`](UIGameTutorialVideoItem.cpp.md) · [`xrEngine/IInputReceiver.h`](../../xrEngine/IInputReceiver.h.md) · [`xrEngine/pure.h`](../../xrEngine/pure.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`GamePersistent.cpp`](../GamePersistent.cpp.md) · [`level_script.cpp`](../level_script.cpp.md) · [`UIGameTutorial.cpp`](UIGameTutorial.cpp.md) · [`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md) · [`UIGameTutorialVideoItem.cpp`](UIGameTutorialVideoItem.cpp.md)
**Tier floor** — T3.

## Purpose

One header for four types, because they are one idea. The sequencer's own substance is in
[`UIGameTutorial.cpp`](UIGameTutorial.cpp.md); the two step kinds have their own
implementation twins. What belongs *here* is the **step interface** — what a rebuild must
provide to add a step kind — because that contract is the header's own content and appears
nowhere else.

## `CUISequenceItem` — the step interface

A step is anything that can be started, asked whether it is still playing, asked to stop, and
given input. An implementor must answer:

```text
INTERFACE SequenceStep
  load(document, index)        # read the step's own authored fields
  start()                      # base: call the on-start script hooks and resolve the
                               # per-frame functor. An override must call the base first.
  stop(force) -> bool          # RETURN false to refuse; the sequencer then keeps the step.
                               # base: call the on-stop script hooks and RETURN true
  update()                     # base: call the per-frame script functor with a progress
                               # factor in 0..1 supplied by `current_factor`
  render()
  on_key_press(code) / on_mouse_press(button) / on_controller_press(button)
  is_playing() -> bool         # the sequencer advances when this turns false
  current_factor() -> real     # progress through the step; base returns 1
```

**Contract of the base** — every step, whatever its kind, carries four authored things the
base owns: a set of **disabled actions** (named, resolved to action identifiers at load, and
consulted by `AllowKey`), a list of **on-start** script functions, a list of **on-stop**
script functions, a **precondition** function name the sequencer evaluates before starting
the step, and a **per-frame** function name called with the step's progress factor.

**Invariants** — the disabled set is a set of *actions*, not of keys, so it follows the
player's bindings. `AllowKey` answers whether a raw code's bound action is permitted; a code
bound to nothing is always permitted.

**Notes** — the flag bits are allocated in two ranges: the base owns the low seven, and a
derived kind continues from a named boundary. That is a layout convention a rebuild replaces
with separate fields; what it encodes is that a step's state is *base state plus kind state*,
and the video step's four extra bits are its own.

## `CUISequencer`

The player of a sequence. It is simultaneously a **frame callback**, a **render callback**
and an **input receiver**, and never a dialog screen — see the implementation twin for why.

- `Start(name)` / `Stop` / `Next` / `Destroy` — the lifecycle.
- `MainWnd` — the shared canvas steps attach their own trees into.
- `IsActive`, `Persistent`, `GetTutorName` — interrogation, used by the game layer to decide
  whether a tutorial survives a level change.
- `m_on_destroy_event` — a callback fired when the sequence ends, however it ends. This is
  the sequence's only completion signal.
- `m_pStoredInputReceiver` — public, because a step forwards to it directly.
- the seven flags, listed in the implementation twin.

## `CUISequenceSimpleItem`

A step made of timed overlay elements. Substance in
[`UIGameTutorialSimpleItem.cpp`](UIGameTutorialSimpleItem.cpp.md). Its declaration carries
one record worth naming here:

```text
RECORD SubItem          # one timed overlay element
  widget  : Label
  start   : real (s)    # offset from the step's own start
  length  : real (s)
  visible : bool        # the current state, so show/hide fires on transitions only
```

## `CUISequenceVideoItem`

A step that plays a full-motion video with one or two audio channels. Substance in
[`UIGameTutorialVideoItem.cpp`](UIGameTutorialVideoItem.cpp.md). The video surface itself is
a renderer-provided object, obtained through the render factory — the decode lives behind
[Seam: Audio and video codecs](../../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs).
