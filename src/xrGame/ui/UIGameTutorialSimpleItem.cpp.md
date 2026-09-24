# src/xrGame/ui/UIGameTutorialSimpleItem.cpp

> One tutorial step made of overlay elements that appear and disappear on their own timeline, optionally holding the game paused, opening a named PDA page, pinning the cursor, and refusing to end until the player performs a named action.

**Needs** — [`UIGameTutorial.h`](UIGameTutorial.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIGameSP.h`](../UIGameSP.h.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UIActorMenu.h`](UIActorMenu.h.md) · [`UITalkWnd.h`](UITalkWnd.h.md) · [`MainMenu.h`](../MainMenu.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`UIGameTutorial.h`](UIGameTutorial.h.md)
**Tier floor** — T3.

## Purpose

The workhorse step. Its authored content is a widget tree plus a **timeline**: a list of
sub-elements each with a start offset and a duration, relative to the step's own start. The
step also carries the four things a tutorial needs to reach out of itself — pause the world,
show a particular PDA page, put the cursor somewhere, and wait for the player to do
something.

## State

```text
RECORD SimpleStep EXTENDS SequenceStep
  root           : Window            # attached into the sequencer's canvas while live
  subitems       : list<SubItem>     # the timeline
  sound          : optional<Sound>
  start_time     : real (s)          # see the deferral protocol below
  length         : real (s)          # 0 means "no timeout"
  pda_page       : text              # empty means "do not touch the PDA"
  cursor_pos     : point             # (0,0) means "do not move the cursor"
  guard_action   : Action            # the action that permits this step to end
  actions        : list<(Action, script function, finalizes?)>
  pda_time_dilation_was : bool       # restored on stop
```

**Invariants**

- A sub-element is found in the tree **by the name `auto_static_<n>`**, where `n` is the
  element's index in the document. Those names are frozen against every shipped tutorial
  document. A missing one is reported and skipped rather than faulting, so a partly-edited
  document still runs.
- `length` of zero means the step never times out on its own and must be ended by the guard
  action or by the sequence being stopped.
- A step whose guard action is *unbound* can stop immediately — the guard degrades to no
  guard rather than to an unpassable one.

## The deferred start

**Contract** — the step's clock does not start when the step starts. It starts on the
**second render** after that.

```text
start():     start_time := -3
first render:  since start_time < -2, start_time := -1
second render: since start_time < 0, start_time := now
is_playing():  RETURN true while start_time < 0, else now < start_time + length
```

**Notes** — the two-frame deferral exists because starting a step can trigger a level load, a
PDA open or a script call that costs an arbitrary amount of wall time. A step timed from its
start call would lose its first seconds to that hitch. Waiting for two frames to have been
*drawn* is a proxy for "the frame time is real again". The sentinel encoding — minus three,
minus one, then the real time — is a state machine in one number and a rebuild should use an
explicit one.

The progress factor the base passes to the step's per-frame script hook is
`(now - start) / length`, or zero while the clock has not started or the length is zero.

## The timeline

**Contract** — each update, every sub-element's window `[start + offset, start + offset +
length]` is compared to the current time, and the element is shown or hidden **only on a
transition**. Showing also restarts the element's colour animation, so an element that
reappears fades in again.

```text
FUNCTION update_timeline()
  now := current time
  base := start_time if the clock has started, else now
  FOR EACH sub IN subitems
    playing := now > base + sub.start AND now < base + sub.start + sub.length
    IF playing AND NOT sub.visible THEN show sub; restart its colour animation
    IF NOT playing AND sub.visible THEN hide sub
```

**Notes** — using `now` as the base before the clock starts means the whole timeline slides
with the deferral rather than starting without it. The comparison is inclusive by an epsilon
at both ends so an element with zero-length overlap still fires.

## Yielding to game screens

**Contract** — a step that does **not** name a PDA page hides its whole overlay whenever the
inventory, the PDA, the conversation screen or the level-change screen is up, and whenever
the main menu is active and the sequence is not marked as drawing over the main menu. It
shows again when they close.

**Notes** — the sequencer draws over everything, which is right for a hint and wrong for a
hint that would sit on top of the inventory the player just opened. This is the correction,
and it is *conditional on the step not being about the PDA* — a step that deliberately opened
a PDA page must stay visible over it, which is the whole point of that step kind.

## Widescreen fitting

**Contract** — each sub-element's width is corrected for the non-uniform canvas stretch, and
the correction differs by game:

```text
IF the mounted game is the first one          THEN no correction
ELSE IF the mounted game is the second one    THEN IF widescreen, divide width by 1.2
ELSE                                          multiply width by the runtime horizontal factor
```

and, independently, a sub-element may carry an authored **widescreen rectangle** which, when
present and the display is wide, replaces its position and size outright.

**Notes** — three generations of the same fix, all still present because all three games'
data is still loaded. The first game's tutorials were authored for 4:3 and are left alone;
the second's were corrected by the fixed 1.2 ratio; the third's are corrected by the actual
runtime factor. The authored widescreen rectangle overrides all of it and is what the last
game's data actually uses. A rebuild must keep all four paths or one of the three games'
tutorials is laid out wrong. This is the second of the two places chapter 15 says the
non-uniform stretch is corrected.

## Reaching into the PDA

**Contract** — a step may name one of a fixed set of PDA pages. Naming one opens the PDA at
that page; naming none, while the PDA is open, closes it. Either way the PDA's own **time
dilation** is disabled for the duration and restored on stop.

**Notes** — the page names are a closed vocabulary shared with the shipped tutorial documents
— map, tasks, faction war, statistics, ranking, logs, and the secondary task panel — and they
are matched case-insensitively against strings in the document. Disabling the time dilation
matters because the PDA normally slows the world while it is open, and a tutorial timed in
seconds would run at the wrong rate.

Opening and closing use the screen's own toggle, so the PDA goes through its ordinary
open/close protocol rather than being forced.

## The guard and the action table

**Contract** — two distinct things react to a key press:

```text
FUNCTION on_press(code)
  # 1. the guard: may this step end yet?
  IF the step is still guarded
    IF the guard action is unbound        THEN unguard      # degrade, do not deadlock
    IF the guard is "any", or the code is bound to the guard action THEN unguard
  # 2. the action table: run a script function per matching action
  FOR EACH (action, function, finalizes) IN actions
    IF the code is bound to that action
      call the script function
      IF finalizes THEN unguard; discard the on-stop hooks; stop the step
```

**Notes** — the guard is what "press ENTER to continue" is: the step declares an action and
refuses to stop until it has been seen. `any` matches any bound action, which is the
"press any key" case, expressed as a sentinel action rather than a separate flag.

A *finalizing* action **clears the step's on-stop script hooks before stopping**. That is
deliberate and easy to miss: the action's own function has already done whatever the stop
hooks would have done, and running both would do it twice.

Mouse and controller presses are routed into the same handler, so a guard action bound to a
mouse button or a pad button works identically. The key press handler is the only one; there
is no separate mouse path.

## Pause handling

**Contract** — a step's pause policy is the same three-valued "on"/"off"/ignore the sequence
carries, and it is applied and undone symmetrically against the pause state recorded at the
step's start. A step that pauses also **pauses sound separately**, which the sequence-level
policy does not.

**Notes** — pausing the world and pausing sound are two independent switches in the engine,
and a tutorial that freezes the world while a voice-over keeps playing is the case that
motivated splitting them. The pause reason string is passed through so the engine's log names
who paused.
