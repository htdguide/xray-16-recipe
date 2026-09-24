# src/xrGame/queued_async_method.h

> Serialises calls to one asynchronous method: while a request is in flight a second request is held, and only the newest held request runs next.

**Needs** — _(none beyond the delegate type its user supplies)_
**Used by** — [`account_manager.cpp`](account_manager.cpp.md) · [`account_manager.h`](account_manager.h.md) · [`login_manager.h`](login_manager.h.md)
**Tier floor** — T3: a one-slot queue around a callback

## Purpose

The account and login managers expose operations that take a network round trip and report
through a callback: *log in*, *fetch my profiles*, *check whether this e-mail is taken*,
*suggest a nickname*. The user interface fires these from keystrokes and button presses, so
a second request routinely arrives while the first is still outstanding. Letting both run
means two callbacks racing to update one screen; refusing the second means the screen
ignores the user.

This file resolves that: at most one request is in flight, at most one is pending, and a
new request **replaces** the pending one. The user's most recent intent wins and everything
older is dropped. Cancellation is the same mechanism with an empty pending request.

It is generic because four managers need it with four different argument lists, and it
predates the language having tuples — hence the hand-written fixed-arity parameter bundles
at the bottom of the file.

## State

```text
RECORD QueuedAsyncMethod
  current_target   : object        # who the in-flight call was made on
  current_args     : ParameterBundle
  current_callback : delegate      # the caller's callback; empty means idle
  pending_active   : bool          # a request (or a cancellation) is waiting
  pending_target   : object        # none means "cancel, do not run anything next"
  pending_args     : ParameterBundle
  pending_callback : delegate
  proxy            : delegate      # bound to this object's own completion handler
```

**Invariants**

- `current_callback` non-empty **is** the definition of "busy". There is no separate flag.
- The underlying method is never called with the caller's callback. It is always called
  with `proxy`, so completion reaches this object first and the queue decides what happens
  next. That indirection is the whole mechanism.
- At most one pending request exists. A third request overwrites the second without it ever
  running — deliberately, since only the newest intent matters.

## `execute`

**Contract** — issues a request, or queues it. Returns immediately either way. Does not
copy the target's lifetime; the caller must outlive the request.

```text
FUNCTION execute(target, args, callback)
  IF busy THEN
    pending_target = target ; pending_args = args ; pending_callback = callback
    pending_active = true
    RETURN
  pending_active = false
  current_target = target ; current_args = args ; current_callback = callback
  invoke the wrapped method on current_target with (current_args, proxy)
```

## Completion — the proxy

**Contract** — runs when the wrapped method finishes. Two paths, and the difference is the
load-bearing part:

```text
FUNCTION on_complete(result_a, result_b)
  IF pending_active THEN
    # The caller no longer wants this answer. Drop the callback FIRST, so that
    # the release hook — which frees whatever the operation produced — cannot
    # observe a callback that is about to be replaced.
    current_callback = empty
    invoke the wrapped RELEASE method on current_target with (result_a, result_b)
    IF pending_target is set THEN execute(pending_target, pending_args, pending_callback)
    RETURN

  current_callback(result_a, result_b)
  current_callback = empty
```

**Invariants** — a superseded result is **released**, not delivered. That release hook is
supplied alongside the method by whoever instantiates the queue, and exists because these
operations hand back allocated payloads (a profile list, a nickname list) that nobody will
free if the callback is skipped.

## `stop`

**Contract** — cancels the in-flight request's *delivery*. Marks a pending slot with no
target, so completion releases the result and starts nothing. Does not reach the network:
the request still completes, its answer is simply discarded. Asserts that something is in
flight.

## `is_active` · `reexecute`

**Contract** — `is_active` reports whether a callback is outstanding. `reexecute` re-issues
the *same* request with the same arguments; it is how a caller retries after a transport
failure without rebuilding the request.

## Parameter bundles

**Contract** — five record shapes holding zero to four values, each comparable for
equality. They exist so the queue can store a request's arguments without knowing what they
are. A rebuild in any language with tuples or variadic generics deletes all five.

**Notes** — the equality operators were written for a de-duplication check (identical
request while identical request in flight → do nothing) that is commented out in the
shipped code. As it stands, re-requesting the same thing re-issues it. Reviving the check
is a behaviour change: a user hammering a button would stop re-sending.
