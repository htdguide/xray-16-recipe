# src/xrGame/secure_messaging.h

> Declares the multiplayer message obfuscation: a seed source, a derived variable-length key, and a symmetric transform over a buffer.

**Needs** — [`Common/Noncopyable.hpp`](../Common/Noncopyable.hpp.md) · [`xrCore/_random.h`](../xrCore/_random.h.md)
**Used by** — [`Level.h`](Level.h.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`secure_messaging.cpp`](secure_messaging.cpp.md) · [`xrServer.h`](xrServer.h.md) · [`xrServer_secure_messaging.cpp`](xrServer_secure_messaging.cpp.md)
**Tier floor** — T1: transforms a buffer in place, word by word, at a fixed width

## Purpose

Declares the surface implemented in [`secure_messaging.cpp`](secure_messaging.cpp.md).

## Exported units

- **The seed generator** — a random source seeded from the processor's cycle counter,
  producing the seeds from which keys are derived. Non-copyable so that two holders can
  never produce the same seed stream.
- **The key record** — a length and a fixed-capacity array of words. Capacity is 32 words
  and the derived length is always between 16 and 32.
- **`generate_key`** — seed to key, deterministically.
- **`encrypt` / `decrypt`** — in-place transform of a buffer, each returning the checksum
  of the *plaintext* side.
