# src/xrGame/account_manager.cpp

> Creating, deleting and looking up a player's online account: field validation done locally, the rest asked of the matchmaking service and answered by callback.

**Needs** — [`account_manager.h`](account_manager.h.md) · [`login_manager.h`](login_manager.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`queued_async_method.h`](queued_async_method.h.md) · [`mixed_delegate.h`](mixed_delegate.h.md) · [`xrGameSpy/GameSpy_GP.h`](../xrGameSpy/GameSpy_GP.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: string validation and asynchronous request bookkeeping; nothing here touches a device or a byte layout

## Purpose

Multiplayer required an account on a vendor service, and this module is the game's side of
account *administration* — as distinct from signing in, which is the login manager's job.
It creates a profile, deletes one, lists the profiles attached to an email and password,
tells you whether an email is already registered, and asks the service for alternative
nicknames when the one you wanted is taken.

Two things in it survive the service's death and are worth a rebuild's attention. The
first is the local validation: every field is checked here before a request is sent, and
the rules are the vendor's rules, so a rebuild targeting a fresh account service must
either keep them or knowingly change them. The second is the shape of every operation —
request, single in-flight slot, callback, default logging callback — which is how the
whole matchmaking layer in this codebase is written.

The service itself was shut down in 2014. The recipe documents what the game *asks for*;
a rebuild should implement this module against a null service, or against its own.

## State

```text
RECORD AccountManager
  service                : connection            # the matchmaking service handle; required, set at construction
  creation_callback      : callback(bool, text)  # one slot per operation kind, not a list
  deletion_callback      : callback(bool, text)
  profiles_callback      : callback(int, text)
  found_email_callback   : callback(bool, text)
  suggestions_callback   : callback(int, text)

  found_profiles         : list<text>            # nicknames attached to the queried account
  suggested_nicks        : list<text>            # service's alternative unique-nickname offers
  verify_error           : text                  # string-table key naming the last validation failure

  profiles_query         : queued operation      # one in flight + at most one pending; see below
  email_query            : queued operation
  suggestion_query       : queued operation
```

Invariants:

- Every result list is cleared *before* the request is issued, not after it returns, so a
  caller that reads the list mid-flight sees empty rather than the previous answer.
- Each result list is published twice: as owned strings and as a parallel run of
  references into them. The second exists only so the UI can hand the collection to a
  list widget without copying. It is invalidated by anything that touches the first, and
  in a rebuild it should not exist at all.
- A callback slot is cleared *before* it is invoked, by copying it out first. The callback
  frequently starts the next operation, and an operation whose callback slot is still
  occupied trips the manager's own occupancy check.
- `verify_error` is not a message. It is a string-table identifier, so the failure reaches
  the player in their own language; nothing in this module ever renders text.

## `create_profile`

**Contract** — validates all four fields locally, then asks the service to register the
account. If the caller supplies no callback, a logging one is installed so that the
console path still reports. Any validation failure short-circuits: the callback fires
immediately with failure and the string-table key of the offending rule, and no request
is sent. A service-side refusal at submission time also reports through the same callback,
so the caller sees exactly one outcome per call regardless of where it failed.

```text
FUNCTION create_profile(nick, unique_nick, email, password, callback)
  creation_callback = callback IF supplied ELSE log_creation
  IF NOT (verify_nick(nick) AND verify_unique_nick(unique_nick)
          AND verify_email(email) AND verify_password(password)) THEN
    creation_callback(false, verify_error)      # local rejection, no traffic
    RETURN
  result = service.new_user(nick, unique_nick, email, password, on_created, self)
  IF result is an error THEN creation_callback(false, translate(result))
```

**Notes** — validation runs left to right and stops at the first failure, so the player is
told about one problem at a time. That is a deliberate UI decision, not laziness: the
account form highlights one field.

## `verify_unique_nick`

**Contract** — the strictest of the four validators, because the unique nickname is the
account's global identity and the service's own namespace rules apply. Returns whether the
candidate is acceptable and, on rejection, leaves a string-table key naming the rule that
rejected it.

**Invariants** — the rules, in the order they are applied:

```text
FUNCTION verify_unique_nick(candidate) -> bool
  IF candidate is absent or empty          -> reject "no unique nick"
  IF length < 3                            -> reject "too short"
  IF length >= 21                          -> reject "too big"
  IF first character IN "@+:#0123456789"   -> reject "bad first symbol"
  IF candidate contains a space            -> reject "must contain no spaces"
  FOR EACH character
    accept only printable ASCII 34..126, excluding
      ','  '\'  '\''  '%'                  -> otherwise reject "must contain only ..."
  RETURN accepted
```

**Notes**

- The length ceiling is one less than the service's field width, which includes a
  terminator. That is the source of every "is too big" bound in this file: 21 for the
  unique nickname, 31 for the display nickname, 51 for the email, 31 for the password.
  They are not tuning values and a rebuild against a different service replaces all of
  them.
- The excluded characters are the ones that break the service's own wire encoding and
  query syntax: a comma is its list separator, a backslash its escape, an apostrophe and a
  percent sign are its query metacharacters. The exclusion of leading digits and leading
  `@`, `+`, `:` and `#` is the service reserving those forms for namespaced and
  email-style identities.
- The lower bound of three characters is the game's own constant, not the service's.

## `verify_nick`

**Contract** — the display nickname: non-empty and shorter than the service's field width.
Nothing else; the display nickname may contain anything the player can type.

**Notes** — the log line for the too-long case says "empty", a copy-paste error in the
original. The string-table key is correct, so the player sees the right message and only
the developer log is wrong.

## `verify_email`

**Contract** — non-empty, shorter than the service's field width, and containing an `@`
that is neither first nor last and has an alphanumeric character on each side. That is the
whole test — no domain check, no dot requirement.

**Notes** — deliberately permissive. The authoritative test is whether the service accepts
it, and a client-side rule stricter than the server's rejects valid accounts.

## `verify_password`

**Contract** — at least two characters and shorter than the service's field width. Both
failures below the minimum report the same string-table key, so "absent" and "one
character" are indistinguishable to the player.

## `delete_profile`

**Contract** — deletes the currently signed-in profile. Requires a signed-in session:
without one the callback fires immediately with the not-logged-in key and nothing is sent.
On the service's confirmation the local profile object is destroyed as well, so the
deletion is not complete until both sides have dropped it.

**Invariants** — the local profile is torn down *inside the service's success callback*,
never optimistically. A failed deletion must leave the player signed in.

**Notes** — reaching the session through the main menu's login manager makes account
administration depend on the menu being alive. That is a service-locator shortcut; a
rebuild should pass the session in.

## `get_account_profiles`

**Contract** — lists the nicknames registered against an email and password. Clears both
result collections, then submits through the single-flight queue. The answer arrives on
the callback as a count, with the names readable from `get_found_profiles`.

**Invariants** — the count handed to the callback and the length of the published
collection are the same number reported twice; a rebuild should hand over the collection.

## `search_for_email`

**Contract** — asks whether an email is already registered, and if so under what nickname.
An empty email is rejected locally. The answer is "found, here is the *first* matching
nickname" or "not found"; additional matches are discarded, because the caller's only
question is whether the address is taken.

## `suggest_unique_nicks`

**Contract** — asks the service for alternatives to a unique nickname that is already
taken. Clears the suggestion collections and submits through the queue. The answer arrives
as a count with the names readable from `get_suggested_unicks`.

**Notes** — the candidate is passed to the service *unvalidated*; suggesting alternatives
for an invalid nickname is a legitimate use, since the point is to find something
acceptable.

## Single-flight operation queue

**Contract** — three of the five operations (profiles, email search, nickname suggestion)
go through a wrapper that permits exactly one request of that kind in flight and remembers
at most one more. A second request while one is outstanding *replaces* any remembered one
rather than queuing behind it, and is issued when the outstanding request completes. Each
operation exposes the same four controls: submit, ask whether one is in flight, re-issue
the in-flight request unchanged, and cancel.

**Invariants** — on submission each operation asserts that its callback slot is empty.
That assertion is the whole reason the queue exists: the service API has one callback
registration per request kind, so two overlapping requests would silently lose one
answer.

- *Re-issue* exists for the case where the underlying connection was re-established: the
  same question is asked again with the same arguments and the same callback.
- *Cancel* does not cancel anything at the service. It marks the in-flight request's
  answer to be dropped on arrival, and the release hook clears the result collections.
  There is no way to withdraw a request once sent.
- The two operations with nothing to release (email search, nickname suggestion) supply
  empty release hooks; only the profile list owns collections that a cancelled request
  must not leave populated.

## Service response handling

**Contract** — five entry points the service calls back into, one per request kind. All
five share one shape: recover the manager from the opaque user pointer handed back with
the response, copy the pending callback out of its slot and clear the slot, translate a
non-success service code into a human-readable description, and invoke the callback
exactly once.

```text
FUNCTION on_service_response(response, manager)
  callback = manager.slot_for_this_kind
  manager.slot_for_this_kind = none          # cleared BEFORE invoking: the callback may resubmit
  IF response.result is an error THEN
    callback(failure, translate(response.result))
    RETURN
  accumulate response payload into the manager's result collections
  callback(success, payload summary)
```

**Invariants** — the account-creation and profile-deletion callbacks are *not* copied out
before invocation, unlike the other three. Those two operations are not queued, so no
resubmission can occur inside their callback; the asymmetry is real and a rebuild should
make all five uniform.

**Notes** — the manager identity travels through the service as an opaque pointer supplied
at request time. That is the C-callback idiom; in a rebuild it is a captured reference and
the recovery step disappears.
