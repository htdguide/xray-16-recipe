# src/xrGame/mixed_delegate_unique_tags.h

> The six names that keep otherwise identical callback slots distinct, one per account operation.

**Needs** — _(none)_
**Used by** — [`mixed_delegate.h`](mixed_delegate.h.md)
**Tier floor** — T4: six constants

## Purpose

Several account and store operations complete with the same shape of answer, so the callback
slots for them would be the same type — and a script binding keyed by type cannot then give
them separate names. Each tag makes one slot a distinct type. See
[`mixed_delegate.h`](mixed_delegate.h.md) for why the mechanism exists.

The value of a tag is never compared, stored or transmitted. Only its distinctness matters.

## State

```text
ENUM MixedDelegateTag
  none                   # the default; two slots that both take it collide
  login_operation
  account_operation
  suggest_nicks
  account_profiles
  found_emails
  store_operation
```

**Notes** — the list is a census of the account seam's asynchronous operations: signing in,
general account changes, nickname suggestion, profile lookup, email search, and a store
purchase. It is therefore a useful summary of what that seam was asked to do, which is worth
more than the tags themselves — the matchmaking service behind it no longer resolves.

The default value is documented in the source as producing a compiler error when two slots
share it, which is the mechanism working as intended: the error is the reminder to add a tag.
A rebuild whose script binding registers by name has nothing here to carry over.
