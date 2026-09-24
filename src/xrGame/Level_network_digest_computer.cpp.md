# src/xrGame/Level_network_digest_computer.cpp

> Hashes the machine's retail product key and sends the hash to the server, so the server can tell two clients apart without ever seeing the key.

**Needs** — [`Level.h`](Level.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [`Level_secure_messaging.cpp`](Level_secure_messaging.cpp.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: read a string, hash it, send it

## Purpose

Multiplayer servers of the era identified a client by its retail product key, but sending
the key itself would let any server operator harvest working keys. The compromise is a
digest: the client hashes its key and sends the hash, which is stable per installation and
useless to a thief. This is the entire content of the file, and it sits behind the dead
matchmaking seam — a rebuild that does not reimplement key-based identity can replace it
with any stable per-installation identifier, or with nothing.

## State

`Stateless.` The computed digest is stored on the level, not here.

## `ComputeClientDigest`

**Contract** — reads the product key from the platform's registry-equivalent store,
upper-cases it, hashes it, and writes the printable hash into the caller's buffer. Returns
that buffer. When no key is installed it writes an empty string — the absence of a key is
not an error here; the server decides whether to accept an empty digest.

```text
FUNCTION ComputeClientDigest() -> text
  key = read product key from the platform key store      # at most 64 bytes
  IF key is empty THEN RETURN ""
  key = uppercase(key)         # the key store's case is not canonical; the digest must be
  RETURN printable_hash(key)   # 32 hex characters
```

**Notes** — the hash is MD5, which is not a security choice and should not be read as one:
nothing here resists an attacker who has the key, and the hash is reversible by dictionary
attack against a key space that small. Its job is to avoid casually transmitting the key,
not to protect it. A rebuild should keep the *shape* (a stable, non-reversible-looking
identifier) and is free to choose a modern hash, at the cost of not matching an original
server's records.

Uppercasing before hashing is load-bearing: two installations with the same key stored in
different case must produce the same digest.

## `SendClientDigestToServer`

**Contract** — computes the digest, remembers it on the level, and sends it to the server
over the encrypted message channel, reliably and in order. One-shot, called during the
connection handshake.

**Notes** — it goes out through the secure send path rather than the ordinary one
(see [`Level_secure_messaging.cpp`](Level_secure_messaging.cpp.md)). That is the only
protection the digest gets in transit.
