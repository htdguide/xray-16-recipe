# src/xrGame/account_manager.h

> Declares the online-account administration surface implemented in [`account_manager.cpp`](account_manager.cpp.md).

**Needs** — [`mixed_delegate.h`](mixed_delegate.h.md) · [`queued_async_method.h`](queued_async_method.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`MainMenu.cpp`](MainMenu.cpp.md) · [`account_manager.cpp`](account_manager.cpp.md) · [`account_manager_console.cpp`](account_manager_console.cpp.md) · [`account_manager_script.cpp`](account_manager_script.cpp.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`login_manager.h`](login_manager.h.md) · [`ui_export_script.cpp`](ui_export_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the account manager and the four callback shapes its operations answer through:
a success flag with a description, a count with a description, and two more that are
structurally identical to those. Substance is in
[`account_manager.cpp`](account_manager.cpp.md).

Exported units:

- `create_profile`, `delete_profile` — account lifecycle.
- `get_account_profiles` / `is_get_account_profiles_active` / `reinit_get_account_profiles`
  / `stop_fetching_account_profiles` — list the nicknames on an account, with the
  submit / in-flight / re-issue / cancel quartet every queued operation exposes.
- `search_for_email` and its quartet — is this address registered.
- `suggest_unique_nicks` and its quartet — alternatives for a taken nickname.
- `verify_unique_nick`, `verify_email`, `verify_password`, `get_verify_error_descr` — the
  local validators and the string-table key of the last failure; exported because the
  account-creation screen validates as you type, before any request.
- `get_found_profiles`, `get_suggested_unicks` — the last answers.
- A record describing a profile to be created (display nickname, unique nickname, email,
  password) — declared for the UI's form, not consumed by the manager, which takes the
  four fields directly.

**Notes**

- The manager is non-copyable by declaration. The real constraint is that it owns one
  callback slot per operation against a service connection it does not own; duplicating it
  would produce two objects racing for the same slots.
- The class is exported to the script layer, so its operation names are part of the frozen
  script surface even though the service behind them no longer answers.
- The service's connection type is forward-declared as an opaque handle rather than
  included, explicitly to keep the vendor SDK's headers out of the game's translation
  units. A rebuild gets this for free by putting the service behind an interface.
