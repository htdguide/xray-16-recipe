# src/xrGameSpy/GameSpy_GCD_Client.cpp

> The client half of product-key authentication: turn a challenge into a response, without
> ever putting the key itself on the wire.

**Needs** — [`GameSpy_GCD_Client.h`](GameSpy_GCD_Client.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_GCD_Client.h`](GameSpy_GCD_Client.h.md)
**Tier floor** — T2: one transformation over strings.

## Purpose

Answers a server's challenge. That is the entire file, and it is worth its own page only
because of what it establishes: the client's product key **never leaves the machine**. The
server sees a response, forwards it to the service, and the service — which knows the keys
— judges it. A rebuild designing a fresh scheme should preserve that property; it is the
reason the server can authenticate without being trusted.

## State

`Stateless.`

## `CreateRespond`

**Contract** — computes a response to a challenge from the local product key, writing it
into the caller's buffer. Takes a flag distinguishing a first authentication from a
re-authentication, because the service computes the two differently and an answer of the
wrong kind is rejected. **Uppercases the caller's key buffer in place** — the key is
compared case-insensitively and the computation is not, so the normalisation has to happen
somewhere; doing it to the caller's buffer is a side effect the caller has to tolerate.
Does not block, does not allocate, touches no network.

```text
FUNCTION make_response(key : text, challenge : text, mode : {first, repeat}) -> text
  RETURN key_response(uppercase(key), challenge, mode)
```

**Invariants** — the response buffer must hold the service's fixed response length; the
call site supplies 128 bytes and nothing here checks.

**Notes**

The key itself is read from the machine's installation settings, not from this module —
see [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) for where it is stored. On a
platform that has no such store, the key reads back empty and the response is computed
over an empty key, which the service would reject. Today it does not get that far.

**Could not recover** — the response function itself. It is the service's, it is
deliberately not documented, and it cannot be reconstructed from this repository. This is
the one place in the chapter where a rebuild cannot reproduce the original's behaviour
even in principle: a client of this engine cannot be made to satisfy a server that still
demanded the original's responses. That is moot — no such server exists — but it means the
authentication design is *replace, not port*.
