# src/xrUICore/Options/UIOptionsManager.cpp

> A name-to-controls map with four group-wide passes over it, plus a set of pending restart flags that are discharged as console commands after an accept.

**Needs** — [`UIOptionsManager.h`](UIOptionsManager.h.md) · [`UIOptionsItem.h`](UIOptionsItem.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md)
**Used by** — [`UIOptionsManager.h`](UIOptionsManager.h.md)
**Tier floor** — T3.

## Purpose

The settings layer between the options screens and the console variables underneath. Every
settings control registers itself here under a group name; the manager then drives all the
controls of a group through the four passes a settings screen needs — load current values,
save edited ones, revert, and restore defaults — without any screen knowing which controls
it holds.

It also carries the deferred half of settings: changes that cannot take effect immediately
set a restart flag, and accepting the screen discharges those flags as console commands in
a defined order.


## State

```text
RECORD OptionsManager
  groups        : map<text, list<OptionsItem>>   # registration order is preserved
  restart_flags : set of { video, sound, ui, system }
```

**Invariants**

- Registration order is the order every pass visits controls in, and it is the order the
  screen built them, which is the XML document order. Two controls in one group that affect
  each other therefore resolve in document order — the shipped pages depend on this.
- A named group that does not exist is a hard error on every pass. A screen that asks for a
  page whose controls were never built aborts rather than silently doing nothing.
- The same control may appear in only one group; unregistration removes the first match found
  and stops.

## `register_item` / `unregister_item`

**Contract** — append to the named group, creating it if needed; or find the control in any
group and remove it. Unregistration scans every group, so it is correct even if the caller has
forgotten which group the control joined.

## The four group passes

**Contract** —

- `set_current_values(group)` — every control reads its console variable.
- `save_backup_values(group)` — every control snapshots itself.
- `save_values(group)` — every control that reports itself changed writes itself back.
- `undo_group(group)` — every control that reports itself changed restores from its snapshot.

**Notes** — the two write passes filter on *changed* and the two read passes do not. That
filter is the whole reason `is_changed_value` has no default implementation: a control that
could not answer would either never be written or always be written, and both are wrong.

## `send_message_to_group`

**Contract** — hands a text message to every control in the group, in registration order. The
message is uninterpreted; the receiving controls agree on its meaning among themselves.

## `options_post_accept`

**Contract** — discharges the pending restarts by executing the corresponding console commands
— graphics restart, sound restart, UI reload — and then clears those three flags. The system
restart flag is deliberately *not* cleared and no command is issued for it: nothing in the
process can restart the application, so the flag stays raised for a screen to read and turn
into a message to the player.

```text
FUNCTION options_post_accept()
  IF video flag THEN console.execute("vid_restart")
  IF sound flag THEN console.execute("snd_restart")
  IF ui    flag THEN console.execute("ui_restart")
  clear video, sound and ui flags        # system flag survives, by design
```

**Notes** — the order is fixed: graphics, then sound, then UI. The UI reload must come last
because it destroys and rebuilds every widget, including the settings screen that is running
this call — so anything scheduled after it would run against released objects.
