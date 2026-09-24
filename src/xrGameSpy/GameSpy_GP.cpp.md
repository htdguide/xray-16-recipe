# src/xrGameSpy/GameSpy_GP.cpp

> The account service: create a profile, find out which profiles an email owns, log in,
> claim a globally unique nickname, delete. Every operation is asynchronous and every one
> of them is a request whose shape a rebuild must reproduce.

**Needs** — [`GameSpy_GP.h`](GameSpy_GP.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_GP.h`](GameSpy_GP.h.md)
**Tier floor** — T2: an asynchronous session with callbacks. Nothing needs manual layout.

## Purpose

This is the account model's transport half. It owns one session with the account service
and exposes the seven operations the game performs against it. It holds no account state
of its own — the game's account and login managers hold that — so this file is best read
as *the request catalogue*: what the game asks an accounts service for.

## The account model

Stated once here, because it is the thing a rebuilder needs and it is spread across this
file and its two consumers in the game module.

```text
RECORD Account                 # what the service stores; the game never holds all of it
  email     : text             # the account's identity for login and recovery. One email
                               # may own several profiles.
  password  : text             # verified by the service; the game never sees it verified
  profiles  : list<Profile>

RECORD Profile                 # what a player plays as
  profile_id   : int           # the service's durable identity for this player
  nick         : text          # display name. NOT unique — several profiles may share one
  unique_nick  : text          # globally unique claim within the namespace; bounded length
  login_ticket : text          # fixed-length bearer token proving this session logged in
  online       : bool
```

**Invariants**

- **Three names, three jobs.** `email` identifies the *account*, `nick` is a display name
  and is not unique, `unique_nick` is the claimed-and-owned name. Logging in needs email
  *and* nick *and* password, because the email alone does not pick a profile. This is the
  single most transferable fact in the chapter: a rebuild that collapses these into one
  name breaks the login screen's whole flow, which is *enter email and password → fetch
  the list of nicks this account owns → pick one → log in*.
- A profile may exist **without** a unique nick. The login flow detects that and sends
  the player to a claim-a-name screen before letting them play; that is why
  `SetUniqueNick` exists as a separate operation from profile creation.
- The login ticket is fixed-length and is fetched *after* a successful login, not returned
  by it. It is the credential the statistics channel presents (see
  [`GameSpy_ATLAS.cpp`](GameSpy_ATLAS.cpp.md)).
- The namespace id (1, "global") scopes unique-nick claims. Two namespaces could both
  hold the same unique nick. A rebuild with one namespace drops the concept.

## State

```text
RECORD AccountSession
  connection : SessionHandle    # none until initialised; one per process
```

## `Init` · construction

**Contract** — opens a session against this title's product id and the global nickname
namespace, and installs the error callback. Returns whether it succeeded. Construction
calls it and ignores the result, so a failed session leaves the object alive with a null
handle; every later operation then fails at the call, and the poll logs a line per frame
(see the note under `Think`).

## `Think`

**Contract** — one poll per frame. Delivers completed operations to their callbacks. Logs
and returns when the session was never established.

**Notes** — the "not initialised" log has no rate limit, so on the dead-service path it
emits once per frame for as long as the main menu is open. A rebuild should report it
once. This is a real symptom of the null path, not a hypothetical.

## `Connect` — log in

**Contract** — logs in with `(email, nick, password)`, non-blocking, completing through
the caller's callback. Declares the client as *behind a firewall*, which asks the service
to keep the session on a connection the client opened rather than expecting to be called
back.

```text
FUNCTION log_in(email, nick, password, on_done) -> submitted_or_error
```

**On completion**, the game's login manager does three more things before it considers the
player logged in, and a rebuild's login must do the equivalent:

```text
1. fetch the login ticket for this session
2. build the local profile record from (profile id, unique nick, ticket)
3. authenticate the same credentials a second time, against the statistics service,
   to obtain the certificate and private data that authorise statistics submission
   (see GameSpy_ATLAS.cpp)
```

Step 3 failing discards the profile and reports the login as failed. So a login is not
one round trip but two, against two services, with the second able to veto the first.

## `NewUser` — create a profile

**Contract** — creates an account and its first profile from `(nick, unique_nick, email,
password)` in one call, non-blocking. The unique nick is claimed as part of creation, so
this can fail for a reason the player must be able to act on — the name is taken — which
is what `SuggestUNicks` exists to soften.

## `GetUserNicks` — which profiles does this email own

**Contract** — given `(email, password)`, yields the list of nicknames the account owns.
Non-blocking. This is the second step of the login screen and the only way the client
learns which nick to log in with. Its failure is also how the client learns an email is
unknown or a password wrong, *before* attempting a login.

## `ProfileSearch` — does this account exist

**Contract** — searches for profiles by any of `(nick, unique_nick, email)`, non-blocking,
unpaginated. The game uses it for exactly one thing: telling a player at the
registration screen that their email is already registered.

## `SuggestUNicks`

**Contract** — given a desired unique nick, yields alternatives that are free.
Non-blocking. Purely a courtesy, but it is a *service* courtesy — the client cannot
compute it, because only the service knows what is taken.

## `SetUniqueNick`

**Contract** — claims a unique nick for the logged-in profile, non-blocking. Asserts the
name is within the bounded length before submitting. On success the local profile's
unique nick is replaced. This is the operation that completes a profile that was created
without a claimed name.

## `DeleteProfile`

**Contract** — deletes the logged-in profile, non-blocking. The account and its other
profiles survive.

## `Disconnect`

**Contract** — ends the session. Synchronous, no callback, no failure path.

## `ShutDown` · destruction

**Contract** — destroys the session if one exists. Does not disconnect first.

## `TryToTranslate`

**Contract** — maps an operation's failure into a **localization key**, not a message.
Four causes get their own key — out of memory, bad parameters, network failure, server
failure — and everything else becomes a key with the numeric code appended, which the
string table will not have an entry for.

**Notes** — the numeric fallback producing a key that resolves to nothing is the shipped
behaviour: an unusual failure shows the player the key itself. That is a bug a rebuild
should fix by falling back to a generic message. The design decision worth keeping is the
one above it: **failures are reported as localization keys, so the account layer never
composes player-facing text.** Contrast the availability probe, which does compose it, and
is the inconsistency.

## `OnGameSpyErrorCb`

**Contract** — session-level errors that belong to no particular operation arrive here and
are logged with their code and description. A *fatal* error is logged and otherwise
ignored — the session is not torn down and no operation is cancelled.

**Notes** — treating fatal as equivalent to non-fatal is the shipped behaviour and means a
dead session stays nominally alive, failing operation by operation. A rebuild should let a
fatal session error invalidate the session so callers fail fast.

## The null path

With the service gone: the session either fails to open or opens against nothing, every
operation either refuses at the call or never completes, the login screen's "fetch my
nicks" step hangs until its own timeout, and no player is ever logged in. The game's
account layer has one graceful consequence of this worth naming — an *offline* login path
exists that takes a nickname alone, builds a local profile with no profile id, no ticket
and `online = false`, and lets the player into multiplayer. **That is the null path for
accounts**: a rebuild that ships no account service at all, and makes every player an
offline profile named by a local setting, loses nothing the engine currently provides.
What it loses is what the online profile carried: a durable profile id, a globally unique
name, and the ticket that authorises statistics.
