# src/xrPhysics/PHCommander.cpp

> Evaluating a list that the evaluation itself may modify, destroy entries of, or
> throw out of.

**Needs** — [`PHCommander.h`](PHCommander.h.md) · [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`PHReqComparer.h`](PHReqComparer.h.md)
**Used by** — [`PHCommander.h`](PHCommander.h.md)
**Tier floor** — T3: list management. The only demand on the tier is a lock.

## Purpose

The list management behind [`PHCommander.h`](PHCommander.h.md). Almost all of it is
consequence of one fact: **an action may reach back into the commander while the commander
is running it**. Scripts add and remove calls from inside callbacks; an object destroyed by
an action removes all of its own calls; an action may add the call that replaces it.

Stateless beyond the two collections declared in the header.

## `update`

**Contract** — one evaluation pass. Applies everything queued since the last pass, then
evaluates every call in insertion order, running each action whose condition holds and
destroying each call that has become obsolete. Does not block; does not allocate except when
a call is added during the pass.

```text
FUNCTION update()
  apply the deferred queue                 # adds appended, removals deleted, queue cleared
  i = 0
  WHILE i < number of calls
      attempt
          calls[i].check()                 # condition, then action if it holds
      on failure
          remove and destroy calls[i] ; CONTINUE without advancing
      IF calls[i].obsolete()
          remove and destroy calls[i] ; CONTINUE without advancing
      i = i + 1
```

**Invariants** — the pass walks by **index, not by a cursor into the collection**, and
re-reads the collection's length every iteration. An action may append to the list or erase
from it, which invalidates any cursor; an index survives an append (the new call is
evaluated this same pass, at the end) and survives an erase ahead of the current position.
An erase *behind* the current position shifts the remaining calls down and one call is
skipped for this pass — accepted, because it will be evaluated next step.

Removal does not advance the index, because the next call has moved into the slot just
vacated.

**Notes** — a call that fails while being evaluated is removed and destroyed rather than
retried. This is a containment decision, not error handling: the commander cannot know what
a foreign action was in the middle of, and a call that failed once at a fixed timestep will
fail again sixty times a second. Removing it keeps one broken script call from taking the
world with it. A rebuild should record which call was dropped — the shipped code does not,
and a silently vanishing physics call is a genuinely hard thing to diagnose.

## the deferred queue

**Contract** — `AddCallDeferred` builds a call and queues it as an addition;
`RemoveCallsDeferred` walks the live list for calls matching a comparer and queues each as a
removal. Neither touches the live list. Both are applied at the top of the next `update`,
additions appended and removals found, destroyed and erased.

**Invariants** — a queued addition owns its call until the queue is applied; if the
commander is cleared first, the queue must destroy the pending additions and must *not*
destroy the pending removals, whose calls still belong to the live list. Getting that
backwards is a double free on shutdown.

**Notes** — the queue exists for the callers who cannot tolerate the immediate forms' hazard
at all: an object being destroyed inside an action removing its own calls, one of which is
the currently executing call. Immediate removal there destroys the action that is running.
Deferral moves the destruction to a point where nothing is executing.

The queue is keyed by the call itself with a flag saying add or remove, so the same call
cannot be queued twice for the same operation. It is not ordered, so an add and a remove of
different calls in the same window are applied in an unspecified order — harmless, because
they address different calls, but a rebuild using an ordered queue loses nothing.

## finding and removing

**Contract** — `find_call` and `has_call` locate the first call whose condition and action
both match their respective comparers. `remove_call` in its comparer form removes every such
call; `remove_calls` removes every call whose condition *or* action matches a single
comparer. `add_call_unique` adds only when no matching call exists, and reports whether it
did.

**Notes** — `add_call_unique` is how a repeating effect avoids stacking. The game layer asks
for "a constant force on this object" every time the condition that produces it recurs, and
without the uniqueness check the object would accumulate one force call per request. Note
that it compares by *comparer*, not by value: two requests for different force magnitudes on
the same object are the same call, and the first one wins.

## the thread-safe forms

**Contract** — `add_call_threadsafety`, `remove_calls_threadsafety` and
`update_threadsafety` take a lock around their plain counterparts.

**Invariants** — the lock protects the live list only against callers on other threads. It
does **not** make the list re-entrant: an action running inside `update_threadsafety` that
calls `add_call_threadsafety` is on the same thread and either deadlocks or re-enters,
depending on the lock's nature. Actions must use the plain or deferred forms. The two
families are not interchangeable and the naming does not say so.

**Notes** — only some call sites use the locked forms, which means the safety is a property
of the caller rather than of the commander. A rebuild should pick one discipline: either the
commander is internally synchronized for every operation, or it is single-threaded and the
game layer marshals. The middle ground here is the least defensible arrangement of the
three.

## `clear`

**Contract** — destroys every live call and every *pending addition*, then empties both
collections. Pending removals are left alone — their calls were already destroyed with the
live list.
