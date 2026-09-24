# src/xrUICore/Options/UIOptionsItem.cpp

> Binds a widget to one console variable by name, reads and writes it through typed console accessors, and records what kind of restart committing the change will cost.

**Needs** — [`UIOptionsItem.h`](UIOptionsItem.h.md) · [`UIOptionsManager.h`](UIOptionsManager.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIOptionsItem.h`](UIOptionsItem.h.md)
**Tier floor** — T3.

## Purpose

Two things: the read/write bridge to the console, and the restart bookkeeping.

The bridge is deliberately asymmetric. *Reading* goes through typed console queries that also
report the variable's declared bounds — which is how a slider learns its own range without the
XML restating it. *Writing* goes through the console's command parser: the control formats a
command line and executes it. That means every settings change takes exactly the same path a
player typing into the console would, including the variable's own validation and side
effects, which is the reason it is done this way rather than by writing the variable directly.

## State

```text
RECORD OptionsItem
  entry    : text            # the console variable's name
  depends  : ENUM { nothing, video_restart, sound_restart, ui_restart,
                    system_restart, apply_on_change }
  # the group registry is shared by every OptionsItem in the process
```

**Invariants**

- An item is registered with the group registry by `assign_props` and unregistered by its own
  destruction, so the registry never holds a released control.
- The variable name is the only link to the model. Two controls bound to the same name are
  both valid and will fight; nothing prevents it.

## `assign_props`

**Contract** — records the console variable's name and joins the named group. Both come from
the XML element's attributes. Joining a group is what makes the control participate in the
page's open, accept and cancel.

## The typed accessors

**Contract** —

- *string*: read the variable's string form; write it by executing `name value`.
- *integer*: read the value and, through the same call, the variable's declared minimum and
  maximum; write by executing `name N`.
- *float*: the same, with a floating-point format.
- *boolean*: read as a boolean; write by executing `name 1` or `name 0`.
- *token*: read the variable's current token *name*, or fetch its whole token table — the list
  of (name, identifier) pairs the variable accepts. This is what fills a drop-down.

**Notes** — the bounds come back from the *read*, so a control must read before it can know its
range. Sliders and spin boxes therefore call the read even when they will immediately
overwrite the value. Formatting a float through the default decimal form is what limits the
precision a settings float can hold, which is unlikely to matter and is worth noting only
because a rebuild that keeps the console round-trip inherits it.

## `save_value`

**Contract** — does nothing when the control reports itself unchanged. Otherwise records the
restart this change requires in the shared registry's flag set. Every implementor chains to
this before writing its own value, so the restart is recorded exactly once per changed
control.

```text
FUNCTION save_value()
  IF NOT is_changed_value() RETURN
  MATCH depends
    video_restart  : registry.require_video_restart()
    sound_restart  : registry.require_sound_restart()
    ui_restart     : registry.require_ui_restart()
    system_restart : registry.require_system_restart()
    otherwise      : nothing
```

## `on_changed_opt_value` / `undo_value`

**Contract** — both write the value immediately when — and only when — the control is marked
apply-on-change. `on_changed_opt_value` is called by a control after each user edit, so an
apply-on-change control is live. `undo_value`'s base does the same, which means undoing an
apply-on-change control writes its restored value straight back to the console rather than
waiting for an accept — correct, since there will be no accept.

**Notes** — the two methods having identical bodies is not a coincidence: for an
apply-on-change control "the value changed" and "the value was restored" are the same event.

## `on_message`

**Contract** — the default broadcast hook does nothing. A control overrides it to react to a
message another control in its group sent, addressed by text. The track bar is the shipped
user: a check box broadcasts, and the slider that depends on it reacts.
