# src/xrGame/login_manager_script.cpp

> Exports the multiplayer account session to the script virtual machine, so the main menu can be written in Lua.

**Needs** — [`login_manager.h`](login_manager.h.md) · [`mixed_delegate.h`](mixed_delegate.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

The multiplayer login screens are script, not engine, so the whole session surface has to
reach Lua. Two registrations plus one delegate declaration.

## State

`Stateless.`

## `login_manager::script_register`

**Contract** — registers `login_manager` with its full operational surface: `login`,
`stop_login`, `login_offline`, `logout`, `set_unique_nick`,
`stop_setting_unique_nick`, the four remembered-credential pairs (email, password,
remember-me, nickname), `get_current_profile` and `forgot_password`. No constructor is
exported: the manager is owned by the main menu and scripts only ever receive the existing
one.

Every operation that can fail takes the result callback as its last argument, which is what
makes the script side event-driven rather than blocking — none of these calls returns a
result.

## `profile::script_register`

**Contract** — registers `profile` with two readers only: the unique nickname and whether
the session is online. The account identifier, the login ticket, the certificate and the
private key are deliberately **not** exported. They are credentials; a script that could
read them could send them anywhere.

## Delegate registration

**Contract** — declares the login result callback's script form under the name
`login_operation_cb`, which is what lets a Lua function be passed where a native callback is
expected. Without it every operation here would be callable from script but unable to report
back.
