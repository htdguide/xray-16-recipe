# src/xrGame/account_manager_console.h

> Declares the console commands implemented in [`account_manager_console.cpp`](account_manager_console.cpp.md).

**Needs** — [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrEngine/xr_ioc_cmd.h`](../xrEngine/xr_ioc_cmd.h.md)
**Used by** — [`account_manager_console.cpp`](account_manager_console.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md)
**Tier floor** — T3: declarations plus one help string each

## Purpose

Declares the nine account-related console commands, each as a command type carrying its
name, whether an empty argument line is acceptable, and its one-line help text. The help
text is the argument spelling's authority and is inlined here rather than in the
implementation; the execution bodies are in
[`account_manager_console.cpp`](account_manager_console.cpp.md).

Exported units, in the order declared:

- create account · list account profiles · sign in · sign out · print current profile ·
  suggest unique nicknames · register a unique nickname · delete current profile · load
  the current profile from the profile store.

**Notes** — five of the nine reach the login manager or the profile store rather than the
account manager, so the file's name understates it: it is the console face of the whole
account layer.
