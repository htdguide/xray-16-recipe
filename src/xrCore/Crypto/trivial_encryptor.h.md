# src/xrCore/Crypto/trivial_encryptor.h

> Declares the archive-header obfuscator and the one process-wide instance of it.

**Needs** — [`trivial_encryptor.cpp`](trivial_encryptor.cpp.md) · [`../xrCore.h`](../xrCore.h.md)
**Used by** — [`trivial_encryptor.cpp`](trivial_encryptor.cpp.md) · [`LocatorAPI.cpp`](../LocatorAPI.cpp.md)
**Tier floor** — T2: a 256-entry permutation and a keystream.

## Purpose

Declares the surface implemented in [`trivial_encryptor.cpp`](trivial_encryptor.cpp.md), and fixes the two key sets as public constants of the type so that a caller can name the region a given archive came from.

## Exported units

- **`trivial_encryptor`** — the obfuscator: construct (which installs the worldwide key), `encode` a buffer, `decode` a buffer, each optionally naming which key to use.
- **`key_flag`** — the two shipped keys: `russian` and `worldwide`. There is no third and no way to supply one.
- **`m_key_russian` / `m_key_worldwide`** — the key triples, readable so that tooling can report which one it matched.
- **`alphabet_size`** — 256; the permutation is over bytes.
- **`g_trivial_encryptor`** — the single shared instance. It carries mutable state (see the twin), which is why the sharing matters.
