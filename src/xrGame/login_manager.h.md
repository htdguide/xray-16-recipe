# src/xrGame/login_manager.h

> Declares the multiplayer account session: the logged-in profile, the two asynchronous operations that establish it, and the stored credentials.

**Needs** — [`login_manager.cpp`](login_manager.cpp.md) · [`account_manager.h`](account_manager.h.md) · [`mixed_delegate.h`](mixed_delegate.h.md) · [`queued_async_method.h`](queued_async_method.h.md) · [`xrGameSpy/xrGameSpy.h`](../xrGameSpy/xrGameSpy.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`MainMenu.cpp`](MainMenu.cpp.md) · [`account_manager.cpp`](account_manager.cpp.md) · [`account_manager_console.cpp`](account_manager_console.cpp.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`login_manager_script.cpp`](login_manager_script.cpp.md) · [`player_account.cpp`](player_account.cpp.md) · [`profile_store.cpp`](profile_store.cpp.md) · [`ui_export_script.cpp`](ui_export_script.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the logged-in `profile` record and the `login_manager` that owns it. See
[`login_manager.cpp`](login_manager.cpp.md) for the substance.

Exported units:

- `profile` — the identity of a logged-in player: account identifier, unique nickname, login
  ticket, whether the session is online, and the certificate and private key pair the
  statistics service issues.
- `login_operation_cb` — the result callback shape shared by every operation here: a profile
  or nothing, plus a string-table identifier describing what happened. The same shape can be
  bound to either a native method or a script function.
- `login_manager` — one at a time: one profile, one in-flight operation.
- `login` · `stop_login` · `login_offline` · `logout` — establish and tear down a session.
- `set_unique_nick` · `stop_setting_unique_nick` — claim a nickname on the account service.
- `save_*_to_registry` · `get_*_from_registry` — the remembered email, password, nickname
  and remember-me flag.
- `get_current_profile` · `delete_profile_obj` — the session, and its explicit teardown.
- `forgot_password` — open a URL in the platform's browser.

**Notes** — both asynchronous operations are wrapped in a queueing helper that serializes
them and supplies the cancel. That helper is what makes "start a login, then cancel it, then
start another" safe; it is documented in
[`queued_async_method.h`](queued_async_method.h.md).
