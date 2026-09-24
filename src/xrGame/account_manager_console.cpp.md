# src/xrGame/account_manager_console.cpp

> The developer console's account commands: sign in and out, create and delete a profile, list and inspect profiles, all routed to the managers the main menu owns.

**Needs** — [`account_manager_console.h`](account_manager_console.h.md) · [`account_manager.h`](account_manager.h.md) · [`login_manager.h`](login_manager.h.md) · [`profile_store.h`](profile_store.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`xrGameSpy/GameSpy_Full.h`](../xrGameSpy/GameSpy_Full.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: argument splitting and delegation

## Purpose

Every account operation the game's screens can perform is also reachable from the console,
so that the account layer can be exercised without the menu. That is the file's whole
reason to exist, and it is a useful pattern to keep: the commands are the *integration
test* for a subsystem that otherwise only runs behind a UI.

Each command does three things and nothing else — split its argument line into fixed
fields, reach the relevant manager through the main menu, and submit the operation with
*no* callback, which makes the manager install its own logging callback and print the
outcome to the console. A rebuild can generate this file from the manager's surface.

## State

`Stateless.` Each command is a registered name with an argument parser and a help line.

## Command surface

**Contract** — each entry is a console command name, a declaration of whether an empty
argument line is meaningful, an argument spelling, and the manager call it makes. Console
command names are frozen: shipped configuration and user settings address them by name.

```text
gs_create_account   <nick> <unique_nick> <email> <password>   -> account manager: create profile
gs_list_profiles    <email> <password>                        -> account manager: list account profiles
gs_login            <email> <nick> <password>                 -> login manager: sign in
gs_logout           (no arguments)                            -> login manager: sign out
gs_suggest_unicks   <unique_nick>                             -> account manager: suggest alternatives
gs_register_unick   <unique_nick>                             -> login manager: claim a unique nickname
gs_delete_profile   (no arguments)                            -> account manager: delete current profile
gs_print_profile    (no arguments)                            -> print the signed-in identity, locally
gs_profile          load                                      -> profile store: load the current profile
```

(The names above are the commands' registration spellings; each command's own help text is
the authority on its argument order, and the console prints that help when a command that
requires arguments is given none.)

**Invariants**

- Commands that take arguments declare that an empty line is *not* handled, so the console
  prints their help instead of running them with empty fields. Commands that take none
  declare the opposite.
- Every command asserts that the main menu and the matchmaking subsystem exist before
  touching them. The account layer is owned by the menu and is absent in a dedicated
  server build, so these commands are only meaningful in a client.
- Submitting with an absent callback is the idiom, not an omission: the manager
  substitutes a logging callback, which is exactly the console's desired behaviour.

**Notes**

- Arguments are split into fixed-size buffers by whitespace, so no account field may
  contain a space. That matches the unique-nickname rule but not the display nickname or
  the password, both of which may legally contain spaces and cannot be set from the
  console. A rebuild should quote-aware split.
- `gs_create_account` assembles a profile record from its four arguments and then passes
  the four arguments individually anyway; the record is dead. A rebuild should pass the
  record.
- `gs_print_profile` reads the signed-in identity locally and sends nothing. It is the one
  command here that still works with the service dead.
- `gs_profile` accepts a subcommand word. Only *load* does anything; the reward and
  best-score subcommands print a notice that the statistics half of the profile layer was
  removed from the engine. The removal is a fact about this fork, not about the original
  game.
