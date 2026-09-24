# src/xrUICore/Options/UIOptionsManager.h

> Declares the registry that groups settings controls by page name and drives open, accept, cancel and broadcast across a whole group at once.

**Needs** — [`UIOptionsManager.cpp`](UIOptionsManager.cpp.md) · [`UIOptionsItem.h`](UIOptionsItem.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md)
**Used by** — [`UIOptionsItem.cpp`](UIOptionsItem.cpp.md) · [`UIOptionsItem.h`](UIOptionsItem.h.md) · [`UIOptionsManager.cpp`](UIOptionsManager.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIOptionsManager.cpp`](UIOptionsManager.cpp.md).

A settings page is not an object. It is a *group name* that every control on the page carries,
and this registry is what turns that name back into a list of controls. That is why a settings
screen never enumerates its own widgets: it names a group and the registry does the walk.

## Exported units

- `CUIOptionsManager` — the registry.
- `RegisterItem(item, group)` / `UnRegisterItem(item)`.
- `SetCurrentValues(group)` — console to widgets, for every control in the group.
- `SaveBackupValues(group)` — widgets to backups.
- `SaveValues(group)` — widgets to console, for the changed ones only.
- `UndoGroup(group)` — backups to widgets, for the changed ones only.
- `SendMessage2Group(group, text)` — broadcast.
- `OptionsPostAccept` — execute the restarts the accepted changes accumulated.
- `DoVidRestart` / `DoSndRestart` / `DoUIRestart` / `DoSystemRestart` — set one restart flag.
- `NeedVidRestart` / `NeedSndRestart` / `NeedUIRestart` / `NeedSystemRestart` — test one.

**Notes** — the restart flags are a bit set in a single byte. There are four kinds and the
system restart is the only one the registry cannot perform itself — it is reported to the
screen, which asks the player to restart the application.
