# src/xrGame/profile_store.cpp

> All that remains of profile loading: confirm somebody is signed in, and report success with nothing attached.

**Needs** — [`profile_store.h`](profile_store.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`login_manager.h`](login_manager.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one lookup and one callback

## Purpose

The original fetched a player's awards and best scores from the account service. The
service is dead, so what survives is the *contract* the shipped scripts still call
against: ask to load, get a yes or a no, and find the award sets empty either way.

Keeping the stub rather than deleting the call is the load-bearing decision — the profile
screen is shipped game data and must still run (conformance criterion 10).

## State

`Stateless` in practice — the store's two sets are never written.

## `load_current_profile`

**Contract** — reports through the completion callback immediately; never blocks, never
reaches the network. Succeeds with an empty message when the login manager exists and holds
a current profile; otherwise fails with the string-table key
`mp_first_need_to_login`, which the caller shows to the player.

```text
FUNCTION load_current_profile(progress_cb, complete_cb)
  # progress_cb is accepted and ignored: there is no progress to report.
  login = main_menu.login_manager
  IF login exists AND login.current_profile exists THEN
    IF complete_cb is set THEN complete_cb(true, "")
    RETURN
  complete_cb(false, "mp_first_need_to_login")     # note: not guarded
```

**Notes** — the success path checks that the callback is set and the failure path does not.
That asymmetry is a latent fault, not a decision; a rebuild should guard both. The failure
message is a *key*, never display text — UI text is always resolved through the string
table.

## `stop_loading`

**Contract** — no effect. A load never stays in flight long enough to cancel.
