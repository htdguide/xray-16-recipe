# src/xrGame/login_manager.cpp

> Signing in to the multiplayer account service: a two-stage online login, an offline stand-in, nickname claiming, and the credentials remembered between runs.

**Needs** — [`login_manager.h`](login_manager.h.md) · [`account_manager.h`](account_manager.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`player_name_modifyer.h`](player_name_modifyer.h.md) · [`secure_messaging.h`](secure_messaging.h.md) · [`RegistryFuncs.h`](RegistryFuncs.h.md) · [`xrGameSpy/GameSpy_Full.h`](../xrGameSpy/GameSpy_Full.h.md) · [`xrGameSpy/GameSpy_GP.h`](../xrGameSpy/GameSpy_GP.h.md) · [`xrGameSpy/GameSpy_ATLAS.h`](../xrGameSpy/GameSpy_ATLAS.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`login_manager.h`](login_manager.h.md)
**Tier floor** — T2: an asynchronous protocol driven by callbacks, plus platform credential storage

## Purpose

Multiplayer requires an identity. This file establishes one, in whichever of three ways the
player asked for: a real login against the account service, an offline session with a
nickname and no account, or a restored session from remembered credentials. It also owns the
one piece of genuinely sensitive state in the codebase — the stored password — and the
decision about how it is kept.

The whole file is a client of a third-party account service. What must survive a rebuild is
the *state machine* and the *invariants*, not the service: any account provider needs the
same two-stage shape, the same single-session rule, and the same offline fallback.

## State

```text
RECORD Profile
  account_id   : int            # zero for an offline session
  unique_nick  : text           # the name other players see
  login_ticket : text           # opaque; proves this session to the game server
  online       : bool           # false: a local identity with no account behind it
  certificate  : bytes          # issued by the statistics service, second stage
  private_data : bytes          # ditto; the signing half

RECORD LoginManager
  profile        : optional<Profile>   # at most ONE session at a time
  pending_result : optional<callback>  # at most ONE operation in flight
  last_email, last_nick, last_password, last_unick : text  # the in-flight request's inputs
  registry_scratch : three fixed text buffers            # returned to callers by reference
```

**Invariant** — at most one profile and at most one in-flight operation, both enforced by
refusal rather than queueing at this level: logging in while logged in is refused with a
"log out first" result, and setting a nickname while an operation is running is refused. The
*queueing* is one layer up, in the helper that wraps both operations — it holds a second
request until the first finishes or is cancelled, which is what makes an impatient player
clicking twice safe.

**Invariant** — the in-flight callback is **taken and cleared before it is invoked**, on
every completion path. The callback typically starts the next operation, and leaving it
installed while calling it would make the next operation see an operation already in flight.
Every completion in this file follows that order; a rebuild must too.

**Invariant** — a profile is created only after the *first* stage succeeds and is destroyed
if the *second* fails. A half-authenticated session must never be observable, because the
statistics credentials it lacks are what the game server checks.

**Invariant** — the three credential getters return a pointer into the manager's own
scratch buffers, which the next call to the same getter overwrites. A rebuild returning
owned strings removes the hazard entirely; it is recorded because the current contract is
"use it before you ask again".

## `login`

**Contract** — begins an online login from an email, a nickname and a password. Returns
immediately; the result arrives on the supplied callback. A caller that supplies no callback
gets a logging one, so an operation is never silently swallowed. The request goes through the
queueing helper, so a second login while one is pending is held rather than refused.

## `login_raw` — stage one

**Contract** — the queued body. Refuses if a session already exists. Otherwise remembers the
three inputs — they are needed again in stage two, which runs on a different callback and
cannot be given arguments — installs the pending callback, and asks the account service to
connect.

A *synchronous* failure from the service (bad arguments, no network stack) clears the pending
callback and reports immediately; an asynchronous one arrives later. Both paths translate the
service's error code into a string-table identifier, so the user interface only ever handles
localized identifiers, never raw codes.

## `login_cb` — stage one completion, into stage two

**Contract** — the account service's answer. On failure, clears and reports. On success:
fetch the opaque login ticket, build the profile, and **immediately start stage two** — a
web request to the statistics service using the same three remembered credentials.

```text
FUNCTION on_connect(result)
  callback = pending          # NOT cleared yet: stage two still needs it
  IF result is an error THEN clear pending ; callback(none, translate(result)) ; RETURN

  ticket = account service's login ticket   # empty on failure, logged, not fatal
  profile = Profile(result.account_id, result.unique_nick, ticket, online: true)
  begin stage two: statistics login with (last_email, last_nick, last_password)
```

**Invariant** — the pending callback is deliberately *not* cleared here, uniquely among the
completions, because stage two is the same logical operation and reports through the same
callback. A failure to obtain the login ticket is logged and the ticket left empty rather
than aborting: the session is usable for everything except joining a ticketed server, and
failing the whole login would be worse.

## `wslogin_cb` — stage two completion

**Contract** — the statistics service's answer. On either an HTTP failure or a rejected
login, reports the translated reason and **destroys the profile**. On success, stores the
certificate and private key into the profile and reports success.

The two stages exist because the account service and the statistics service are separate
services with separate credentials, and a session is only complete when both have accepted
the player. A rebuild against a single-service provider collapses this into one stage and
loses nothing but the ordering.

## `login_offline`

**Contract** — creates a local identity with no account behind it, from a nickname alone.
Refuses while an operation is in flight or a session exists. Validates that the nickname
contains at least one non-whitespace character — an all-space nickname is the one input that
would otherwise produce an invisible player — then passes it through the name sanitizer,
which is the same filter applied to every player name, and reports success synchronously.

**Notes** — the "in flight" check here inspects the pending callback directly rather than
going through the queueing helper, so this operation does **not** queue behind a pending
online login; it is refused. That asymmetry is real: offline login is the escape hatch from
a login that is not completing, and making it wait would defeat it.

An offline profile has account identifier zero, an empty ticket and `online` false. Every
later decision — whether a disconnect is needed at logout, whether a nickname change goes to
the service — branches on that flag.

## `set_unique_nick` · `set_unique_nick_raw`

**Contract** — claims a nickname. Queued the same way as login. Refuses with a "log in
first" result when there is no session, and with an "invalid nickname" result when the name
is empty.

For an **offline** profile the change is purely local: sanitize and store, report success
immediately — there is no service to claim it from. For an **online** profile the name is
registered with the account service and the result arrives asynchronously; only then is the
profile's name updated. Claiming can fail because another account holds the name, which is
the whole reason the operation is asynchronous at all.

## `logout` · `release_login` · `reinit_connection_tasks`

**Contract** — `logout` disconnects from the account service if the session was online, then
destroys the profile. Asserted, not checked, that a session exists.

`release_login` is the *cancellation* path the queueing helper calls when a queued login is
superseded: it logs the session out and then **reinitializes every other outstanding request
against the account service**. That last step is the subtle one. Disconnecting resets the
service connection's state, which silently invalidates unrelated in-flight requests that the
account screen may have started — fetching the account's profile list, searching for an
email, asking for nickname suggestions. Each is re-armed if it was active. Without this, a
player who cancels a login and retries sees three screens that never finish loading.

## `save_password_to_registry` · `get_password_from_registry`

**Contract** — store and retrieve the remembered password in platform credential storage,
**encrypted**, with a key derived from a fixed compile-time seed. Reading returns the empty
string when nothing is stored.

**Notes** — this is obfuscation, not security, and the recipe should say so plainly. The
seed is a constant in the executable, so the key is identical on every installation and
anyone with the binary can recover any stored password. The scheme defends against a casual
reader of the credential store and nothing else. A rebuild should use the platform's own
secret storage — which is what "credential storage" means on every modern system — and keep
this entry only as the reason the stored format looks the way it does.

The read buffer is fixed at 128 bytes against a documented 30-character maximum, and the
decrypted result is copied out with no length check. A stored value longer than the buffer
is a memory error rather than a rejection; a rebuild bounds it.

## `save_email_to_registry` · `get_email_from_registry` · `save_nick_to_registry` · `get_nick_from_registry` · `save_remember_me_to_registry` · `get_remember_me_from_registry`

**Contract** — the remaining remembered values, stored in plain text. Empty values are
refused with a logged error rather than stored, so that a blank field never erases what was
remembered. The nickname goes through the *shared* player-name storage rather than a
login-specific key, because the same name is used for single-player statistics and for
offline multiplayer.

## `forgot_password`

**Contract** — opens a URL in the platform's default browser. Fire and forget; no result.
Platform-specific by necessity — this is the one place the file touches the operating
system's shell. A rebuild needs one "open a URL externally" primitive and nothing more.

**Notes** — the Windows path launches the URL through a command interpreter with the URL
interpolated into the command line. A URL containing shell metacharacters would be
interpreted rather than opened. The URL comes from script, which comes from the game's own
configuration, so it is not attacker-controlled in practice — but a rebuild should hand the
URL to the platform's URL opener directly and never build a command line.

## `only_log_login`

**Contract** — the default result callback, installed whenever a caller supplies none. Logs
the failure reason or a success greeting. Its existence is why no operation in this file has
to handle a missing callback.
