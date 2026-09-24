# src/xrCore/Events/Notifier.h

> A fixed set of event slots with subscribable handlers, safe to unsubscribe from inside a handler.

**Needs** — [`Threading/Lock.hpp`](../Threading/Lock.hpp.md) · [`Threading/ScopeLock.hpp`](../Threading/ScopeLock.hpp.md)
**Used by** — [`ai_space.h`](../../xrGame/ai_space.h.md)
**Tier floor** — T2: it owns handler lifetimes explicitly and releases them at a defined moment, but nothing here touches a layout or a device.

## Purpose

A small publish-and-subscribe bus with a **fixed, compile-time number of event kinds**. Each kind owns an independent list of handlers; firing one kind runs its handlers in subscription order in the caller's thread.

The one interesting problem it solves is **self-unsubscription**: a handler very often wants to remove itself while it is running — a one-shot callback, or a screen that closes in response to the event it was waiting for. Destroying the handler at that moment would free the object whose code is executing. The deferral below is the whole reason this file is not three lines of a list.

## State

```text
RECORD HandlerSlot
  handler   : optional<Handler>   # owned; empty means the slot is free
  destroying: bool                # unsubscribe requested
  executing : bool                # its body is on the stack right now

RECORD EventBus<kinds>
  slots : HandlerSlot[kinds][]    # one growable list per event kind
  lock  : Lock                    # one per kind

# invariant: a handler's identifier IS its index in its kind's list, and stays
#   valid for the handler's life. Slots are never compacted; a freed slot is
#   reused by the next subscription to that kind.
# invariant: destroying AND executing together means "free me the moment my
#   body returns" — the only state in which a slot is neither live nor free.
```

## `subscribe` (`RegisterCallback`)

**Contract** — take ownership of a handler and place it in the first free slot of the given event kind, appending if there is none. Returns the slot's index as the subscription identifier. The event kind must be within the compile-time bound; exceeding it is fatal.

**Invariants** — the bus *owns* the handler and will destroy it. A handler created by the caller and registered here must therefore be created by the same means the bus destroys it with — a coupling the source itself warns about. A rebuild should make the bus the only way to create a handler, which is what the next entry does.

## `create_and_subscribe` (`CreateRegisteredCallback`)

**Contract** — pick the slot first, construct the handler *with its own identifier as a constructor argument*, and register it. Returns that identifier. Preferred over the plain form because the handler now knows how to unsubscribe itself, and because the bus alone controls creation and destruction.

```text
FUNCTION create_and_subscribe(bus, kind, handler_kind, args) -> int
  LOCK bus.lock[kind] DURING
    index := first free slot IN bus.slots[kind]
             OR (the position one past the end, if none is free)
    handler := construct handler_kind WITH (index, args)
    place handler AT index, appending if needed
  RETURN index
```

## `unsubscribe` (`UnregisterCallback`)

**Contract** — mark the slot as being destroyed and, *if its handler is not currently running*, free it immediately. If it is running, the slot is left marked and the free happens when the handler returns. Returns whether this call was the one that initiated the removal — a second unsubscribe of an already-removing slot reports false.

```text
FUNCTION unsubscribe(bus, kind, index) -> bool
  LOCK bus.lock[kind] DURING
    IF index IS OUT OF RANGE OR slot IS free THEN RETURN false
    first := NOT slot.destroying
    slot.destroying := true
    IF NOT slot.executing THEN free slot           # handler destroyed here
  RETURN first
```

## `fire` (`FireEvent`)

**Contract** — run every live handler of one event kind, in slot order, in the calling thread, holding the kind's lock for the whole sweep. A slot already marked for destruction is skipped. A slot that marks itself for destruction while running is freed immediately after it returns.

```text
FUNCTION fire(bus, kind)
  LOCK bus.lock[kind] DURING
    FOR index FROM 0 WHILE index < length(bus.slots[kind])
      slot := bus.slots[kind][index]
      IF slot IS free OR slot.destroying THEN CONTINUE
      slot.executing := true
      run slot.handler
      slot.executing := false
      IF slot.destroying THEN free slot
```

**Invariants** — the iteration is by index, not by a cursor into the list, because a handler may subscribe a *new* handler and grow the list underneath the sweep. A handler subscribed during a sweep into a slot the sweep has not yet reached will be run in this same sweep; one placed behind the cursor will not. That is observable and is worth a rebuilder's deliberate choice.

## Notes

The lock is held across the handler bodies, and the unsubscribe path takes the same lock. A handler that unsubscribes itself therefore re-enters the lock, which requires the lock to be re-entrant — it is. A rebuild using a non-re-entrant lock must restructure: collect the live handlers under the lock, release it, run them, then reacquire to apply the deferred frees.

The "find a free slot" scan is linear over the whole list on every subscription. Subscriptions happen at screen and level transitions, never in the frame loop, so it has never mattered.
