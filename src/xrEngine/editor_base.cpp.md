# src/xrEngine/editor_base.cpp

> The debug overlay's shell: a three-state visibility model, a self-registering tool list, and the menu bar that drives both.

**Needs** — [`editor_base.h`](editor_base.h.md) · [`editor_helper.h`](editor_helper.h.md) · [`editor_base_input.cpp`](editor_base_input.cpp.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`device.h`](device.h.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`editor_base.h`](editor_base.h.md)
**Tier floor** — T3: a list, a state machine and menu calls; the toolkit does the work.

## Purpose

The engine hosts an immediate-mode debug interface, and this is its shell — not any tool,
but the thing tools live in. It owns three decisions: when the overlay is visible and
whether it takes input; which tools exist; and the menu bar through which a person reaches
them.

The whole of it is debug-only in effect and can be omitted from a rebuild (see the overlay
seam). What cannot be omitted, if it is kept, is the *three-state* model — a two-state
show/hide overlay is unusable for the thing it is actually for, which is watching a tool's
numbers while playing.

## The three states

**Contract** — the overlay is in exactly one of three states, and the state decides three
things at once: whether the shell's own windows draw, whether tools draw, and whether the
overlay holds input.

```text
hidden   nothing draws          input belongs to the game
full     shell + tools draw     input belongs to the overlay; windows are opaque and movable
light    only tools draw        input belongs to the game; tool windows are transparent,
                                undecorated, unmovable and take no input
```

**Notes** — *light* is the state the overlay exists for. A tool showing a live graph is
useless if watching it means the game cannot be played, and useless in a two-state design
for exactly that reason. Light mode is a tool rendered as a heads-up display: still drawing,
no longer interactive.

Entering *full* captures input; leaving it for either other state releases it. That is the
only side effect of a transition, and it is the one that matters — see
[`editor_base_input.cpp`](editor_base_input.cpp.md), where capture also releases the
pointer grab so the player's mouse becomes a cursor again.

## `SwitchToNextState`

**Contract** — the single cycle the editor key and a double-click both drive. Not a fixed
rotation: whether light is visited depends on whether any tool is open.

```text
FUNCTION next_state(editor)
  CASE hidden -> full
  CASE full   -> IF any tool is open THEN light ELSE hidden
  CASE light  -> hidden
```

**Notes** — skipping light when no tool is open is the decision. Light with nothing open is
an indistinguishable-from-hidden state that a person pressing the key would have to press
through, and the cycle would feel broken. Asking the tools rather than tracking a flag keeps
the answer true when a tool closes itself.

## Tool registration

**Contract** — a tool registers itself with the overlay when it comes into existence and
unregisters when it goes, so the tool list needs no central table and a tool can live
anywhere in the engine. Registration only appends and sets a dirty flag; the list is sorted
by name lazily, the next time the Tools menu is opened.

**Notes** — sorting is deferred because **a tool registers during its own construction**, when
it cannot yet answer what its name is. Sorting at the point of use is the simplest fix; the
alternative is a two-phase construction, which is worse.

## `OnFrame`

**Contract** — joins the update phase. Draws according to the state, then watches for the
double-click that cycles it.

```text
FUNCTION on_frame(editor)
  CASE state IS full
    push the pointer position and the desired cursor to the toolkit
    keep the platform's text-input mode in sync with what the toolkit wants
    draw the menu bar
    FALL THROUGH
  CASE state IS light
    FOR EACH tool
      tool.on_tool_frame()

  IF a double-click landed on no window
    next_state()
```

**Notes** — the fall-through is the three-state model in one line: full is light plus the
shell.

**A double-click on empty space cycles the state.** It is the discoverable gesture for
someone who does not know the key, and it is guarded by "no window is focused" so that
double-clicking inside a tool does not throw the overlay away.

Every tool draws every frame in both visible states. The tool decides whether to open a
window; the shell only supplies the default window flags, which is how light mode makes
every tool transparent and non-interactive without any tool knowing about light mode.

## `get_default_window_flags`

**Contract** — the window style tools must use to participate in the state model. In full
state: a window with a menu bar. In light state: no navigation, no input, not movable, no
decoration, no background.

**Notes** — this is the whole enforcement mechanism for light mode. A tool that asks for the
default flags becomes a transparent read-out; a tool that hard-codes its own does not. The
contract is by convention, which is acceptable for debug code and would not be otherwise.

## `is_shown`

**Contract** — true when at least one tool is open. Distinct from the overlay being visible:
this asks whether there is anything worth showing.

## `ShowMain` — the menu bar

**Contract** — draws the shell's only interface: a File menu with four entries, a Tools menu
listing every registered tool as a checkbox, and an About menu holding the toolkit's own
demo and metrics windows.

```text
File    Console   toggles the engine console             shortcut: the console action
        Stats     toggles the statistics overlay         shortcut: the scores action
        Hide      -> light state                         shortcut: the editor action
        Close     -> hidden state                        shortcut: the quit action
Tools   one checkbox per registered tool, sorted by name     (non-shipping builds only)
About   the toolkit's demo window (debug builds) and its metrics window
```

**Notes** — each File entry displays the key **currently bound** to the corresponding game
action rather than a hard-coded shortcut, and the entries say so in their own tooltip: the
shortcut works only when no overlay window has focus, because otherwise the key belongs to
whatever is focused. That is honest and is the right way to present a global key in a
windowed overlay.

Showing the *bound* key rather than a fixed one means the menu tells the truth after a
rebind. The engine's console and statistics toggles are surfaced here because they are the
two engine-level things a person opening the overlay most often wants, and they are not
tools.

The Tools menu is compiled out of a shipping build while the shell itself is not, so a
shipping build has an overlay with no tools in it — which is what makes the console entry
the only reason to open it there.
