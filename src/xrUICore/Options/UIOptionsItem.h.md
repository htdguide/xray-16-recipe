# src/xrUICore/Options/UIOptionsItem.h

> Declares the protocol a widget implements to be a settings control — read the value, back it up, commit it, undo it — and the typed accessors that bind it to one console variable.

**Needs** — [`UIOptionsItem.cpp`](UIOptionsItem.cpp.md) · [`UIOptionsManager.h`](UIOptionsManager.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIEditKeyBind.cpp`](../../xrGame/ui/UIEditKeyBind.cpp.md) · [`UIEditKeyBind.h`](../../xrGame/ui/UIEditKeyBind.h.md) · [`UICheckButton.cpp`](../Buttons/UICheckButton.cpp.md) · [`UICheckButton.h`](../Buttons/UICheckButton.h.md) · [`UIComboBox.cpp`](../ComboBox/UIComboBox.cpp.md) · [`UIComboBox.h`](../ComboBox/UIComboBox.h.md) · [`UIEditBox.cpp`](../EditBox/UIEditBox.cpp.md) · [`UIEditBox.h`](../EditBox/UIEditBox.h.md) · [`UIOptionsItem.cpp`](UIOptionsItem.cpp.md) · [`UIOptionsManager.cpp`](UIOptionsManager.cpp.md) · [`UIOptionsManager.h`](UIOptionsManager.h.md) · [`UICustomSpin.h`](../SpinBox/UICustomSpin.h.md) · [`UISpinNum.cpp`](../SpinBox/UISpinNum.cpp.md) · [`UISpinText.cpp`](../SpinBox/UISpinText.cpp.md) · _and 5 more_
**Tier floor** — T3.

## Purpose

Declares the mixin implemented in [`UIOptionsItem.cpp`](UIOptionsItem.cpp.md), and it is an
interface header as much as a declaration: what it demands of an implementor *is* the
contract a rebuild must satisfy.

The whole settings system rests on one idea: **the console variable is the model**. A settings
control does not hold a value; it reads the variable when the page opens, shows it, and writes
it back through a console command when the page is accepted. Backup and undo are the control's
own business. There is no settings database.

## The protocol an implementor must satisfy

```text
set_current_value()   # console -> widget. Called when a page is opened.
save_backup_value()   # widget  -> backup. Called at the same moment.
save_value()          # widget  -> console. Called on accept. MUST chain to the
                      # base, which records any restart the change requires.
undo_value()          # backup  -> widget. Called on cancel. MUST chain to the base.
is_changed_value()    # backup != widget. REQUIRED; there is no default.
```

Only `is_changed_value` has no default: every control must be able to say whether the player
touched it, because the group commit uses that answer to decide whether to write at all.

## Exported units

- `CUIOptionsItem` — the mixin.
- `ESystemDepends` — what a change to this control costs: nothing, a graphics restart, a sound
  restart, a UI reload, a full application restart, or *apply immediately*. The last is not a
  cost but a policy: such a control writes on every change rather than on accept.
- `AssignProps(entry, group)` — bind to a console variable name and join a named group. This
  is what the XML layer calls; the entry and group both come from attributes.
- `GetOptionsManager` — the single process-wide group registry, held as a static member.
- `OnMessage(text)` — a broadcast hook; a control can be told something by name without the
  sender knowing its type. Used for cross-control dependencies inside a group.
- The typed accessors — string, integer with bounds, float with bounds, boolean, token — each
  a read from and a write to the bound console variable.
- `SendMessage2Group(group, text)` — broadcast to a group.
- `OnChangedOptValue` — the apply-on-change hook a control calls after every user edit.

**Notes** — the manager is a static member rather than a singleton object because every
settings control in the process shares one registry; the toolkit has no notion of two
independent settings screens.
