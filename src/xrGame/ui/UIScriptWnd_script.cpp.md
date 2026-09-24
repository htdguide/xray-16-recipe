# src/xrGame/ui/UIScriptWnd_script.cpp

> Exports the script screen base to the script tier: which methods a script may override, which it
> may call, and the fourteen typed child lookups that are the only way a script reaches a widget.

**Needs** — [`UIScriptWnd.h`](UIScriptWnd.h.md) · [`xrUICore/ListWnd/UIListWnd.h`](../../xrUICore/ListWnd/UIListWnd.h.md) · [`xrUICore/TabControl/UITabControl.h`](../../xrUICore/TabControl/UITabControl.h.md) · [`xrUICore/Buttons/UICheckButton.h`](../../xrUICore/Buttons/UICheckButton.h.md) · [`xrUICore/Buttons/UIRadioButton.h`](../../xrUICore/Buttons/UIRadioButton.h.md) · [`xrUICore/MessageBox/UIMessageBox.h`](../../xrUICore/MessageBox/UIMessageBox.h.md) · [`xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIScriptWnd.h`](UIScriptWnd.h.md)
**Tier floor** — T1: reaches into the script machine's value stack and instance representation to
enumerate a script object's fields for the inspector

## Purpose

Two jobs that must live together because both need the script machine's internals.

First, the **export**: the class the scripts know as `CUIScriptWnd`, the methods they may call, the
methods they may override, and the typed accessors that let a script get at a named child with the
right type. Acceptance criterion 10 freezes every name in this file.

Second, the **overridable method bridge**: four methods a script subclass may redefine, each of
which must still be reachable in its un-overridden form, because a script's override usually wants
to do its own work *and then* the base behaviour. Each overridable method therefore appears twice in
the export — as the dispatching form and as the explicit base form.

Third, incidentally, the **inspector view of a script screen**: a script subclass has fields that
live in the script machine, not in this process's object, so the development inspector cannot see
them by walking the widget tree. This file walks the script object's own field table instead.

## State

`Stateless.` Everything here is registration and bridging; the screen's state is in
[`UIScriptWnd.cpp`](UIScriptWnd.cpp.md).

## The exported surface

**Contract** — The class is exported under the name `CUIScriptWnd`, derived from the modal dialog
and the factory-object interface, constructible from script with no arguments. These names are
frozen: every shipped game screen is a script that subclasses this name.

```text
CLASS CUIScriptWnd EXTENDS CUIDialogWnd
  constructor()

  # bindings
  AddCallback(control_name, event, function)
  AddCallback(control_name, event, function, receiver)
  Register(child)
  Register(child, name)
  Load(name)

  # overridable: each is callable, and the base form is reachable from an override
  OnKeyboard(scancode, action) -> bool
  Update()
  Dispatch(command, parameter) -> bool
  NeedCursor() -> bool

  # typed child lookup, one name per control type
  GetButton, GetMessageBox, GetPropertiesBox, GetCheckButton, GetRadioButton,
  GetStatic, GetEditBox, GetDialogWnd, GetFrameWindow, GetFrameLineWnd,
  GetProgressBar, GetTabControl, GetListBox, GetListWnd
```

**Notes** — The fourteen accessors are one function distinguished only by its result type; the
script tier has no generics, so each type gets its own exported name. That is the whole reason the
list is enumerated rather than generated, and a rebuild whose script tier *does* have generics may
collapse them — but must keep the fourteen names callable, because shipped scripts use them.

## `GetControl`

**Contract** — Finds a direct or nested child by name and narrows it to the requested control type.
Returns nothing both when no child carries that name and when the child is of a different type — the
two failures are indistinguishable to the caller, which is why a script that asks for the wrong type
sees "missing control" rather than an error.

```text
FUNCTION GetControl<T>(name) -> optional<T>
  child <- FindChild(name)
  IF child IS none THEN RETURN none
  RETURN narrow child TO T      # none if the child is not a T
```

## The overridable methods

**Contract** — Four methods a script subclass may redefine: the keyboard handler, the per-frame
update, the command dispatcher, and the cursor-needed query. Each has two behaviours a rebuild must
reproduce:

- calling the method on the screen calls the script override if one exists, otherwise the base;
- the script override can reach the base behaviour explicitly, and shipped scripts rely on this.

`NeedCursor` is the one with extra logic rather than a plain forward: the override's answer is
**widened**, not replaced — if the script says a cursor is needed, one is; if it says no, the base's
own answer still decides. The other three replace the base outright.

```text
FUNCTION NeedCursor() -> bool
  IF script_override_says_yes() THEN RETURN true
  RETURN base.NeedCursor()
```

**Notes** — A script subclass that overrides `Update` and forgets to call the base form stops the
whole widget subtree from updating. That is a real, shipped hazard of this design and not a bug to
fix, because screens rely on being able to suppress it.

## The inspector view

**Contract** — Development-only; compiled out of the shipping build. Presents a script screen in the
[debug overlay](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) as its own node labelled with
the script class name, under which two things appear:

- in the **tree**, every field of the script object that is itself a widget, recursed into as a
  widget node — which is how widgets a script created and holds only in script fields become
  visible to the inspector at all;
- in the **detail panel**, every field that is a boolean, a number or a string, rendered read-only.

```text
FUNCTION FillDebugTree(state)
  base.FillDebugTree(state)
  depth <- script_stack.depth              # invariant: restored on exit
  push this screen's script object
  node <- tree node named "<script class name> * LUA"
  IF node is open THEN
    FOR EACH field IN script object's field table
      IF field is a script-machine object AND it is a widget THEN
        widget.FillDebugTree(state)        # recurse as a widget
  pop everything pushed
  ASSERT script_stack.depth == depth
```

**Invariants** — The script machine's value stack must be at exactly the depth it started at when
either inspector function returns. Both functions assert it. This is the single load-bearing
constraint in the file: the script machine is shared with the whole game, and a leaked stack slot
corrupts an unrelated call later, far from the cause.

**Notes** — The field table of a script object is reached through the script machine's
instance representation, which differs between script-machine versions; the source carries a
compatibility shim for one such difference. A rebuild reaching a different script machine solves the
same problem — "enumerate an instance's own fields" — however that machine offers it.
