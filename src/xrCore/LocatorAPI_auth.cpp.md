# src/xrCore/LocatorAPI_auth.cpp

> Folds the mounted filesystem and the loaded configuration into one 64-bit number that two machines can compare to prove they are running the same game data.

**Needs** — [`LocatorAPI.h`](LocatorAPI.h.md) · [`FS.h`](FS.h.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [`xrstring.h`](xrstring.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: byte-exact checksums over file contents and over a serialized configuration image; no device or layout concerns beyond reading whole files.

## Purpose

Multiplayer needs a cheap answer to "are your files the same as mine". This file computes that answer: a single unsigned 64-bit *auth code* derived from the serialized authoritative configuration plus the checksums of every mounted file whose path contains one of a caller-supplied list of *important* substrings. It lives apart from the rest of the locator because it is the only part of the filesystem that is about trust rather than about lookup, and because the whole feature is dead weight in single-player.

The scheme is a fingerprint, not a security measure: the fold is a commutative exclusive-or, so a client that can compute one can also forge one. It detects accidental divergence (a modded config, a replaced texture pack), not a determined cheat.

## State

```text
RECORD AuthOptions
  ignore    : list<text>    # substring patterns; a matching path is skipped entirely
  important : list<text>    # substring patterns; a matching non-empty file is checksummed

# owned by the locator, not by this file:
#   auth_code : int (64-bit)   the fingerprint
#   auth_lock : mutex          serializes computation against readers
```

**Invariant** — `auth_code` is only ever read while no computation is in flight. A reader that observes the value must observe the *completed* fold, never a partial one.

**Invariant** — the fold must be order-independent, because the file list is enumerated in whatever order the mount produced. Exclusive-or gives that for free, and is the reason a stronger, order-sensitive hash was not used.

## `auth_generate`

**Contract** — takes an ignore list and an important list and computes the fingerprint synchronously, blocking the caller until it finishes. Allocates one options record for the duration. No result is returned; the fingerprint is left in the locator's `auth_code` for `auth_get` to read.

**Notes** — the name and the indirection through a heap record are a residue of an earlier design in which this ran on its own thread and the record was the thread's argument. A rebuild should pass the two lists directly and either block or return a future; there is nothing asynchronous left to preserve.

## `auth_get`

**Contract** — returns the current fingerprint. Takes and immediately releases the computation lock first, which is how it waits for an in-flight computation to finish; if none is running the wait is free.

**Notes** — acquire-then-release with an empty body is an idiom for "block until the writer is done", and a rebuild should express it as an explicit wait on completion rather than as a lock dance. The original carries a comment questioning the construct, which is fair: with generation now synchronous it does nothing, and only matters if generation moves back onto a worker.

## `auth_runtime`

**Contract** — the computation itself. Holds the computation lock for its whole duration. Serializes the authoritative configuration set to an in-memory buffer, seeds the fingerprint with that buffer's checksum, then walks every entry in the mounted file table and folds in the checksum of each file that survives the filters. Opening a file that the table lists but that cannot actually be read aborts the walk early, leaving a partial fingerprint — which is itself a useful signal, since such a client will never match a healthy one. Frees the options record before returning.

**Invariants** — the configuration checksum is computed over the *serialized* form, not over the parsed sections, so that key order and formatting are whatever the serializer produces deterministically — the two peers must serialize identically or they will never agree. A file with a recorded real size of zero is never checksummed, which keeps directory placeholders and empty markers from contributing.

```text
FUNCTION compute_auth_code(ignore, important) -> int (64-bit)
  LOCK auth_lock DURING
    buffer <- serialize(authoritative_config)      # the "auth" configuration set,
                                                   # a separate parse from the game's own
    code <- crc32(buffer)

    FOR EACH f IN mounted_files
      IF any pattern IN ignore occurs as a substring of f.path
        CONTINUE
      FOR EACH pattern IN important
        IF f.size_real == 0
          CONTINUE
        IF pattern occurs as a substring of f.path
          reader <- open(f.path)
          IF reader IS none
            RETURN code                            # abort: partial code, deliberately
          code <- code XOR crc32(reader.bytes)     # widened to 64 bits; upper half stays 0
          close(reader)
    RETURN code
```

**Notes** — the fold widens a 32-bit checksum into a 64-bit accumulator and never touches the upper half, so the result is effectively 32 bits wide. That is either an unfinished widening or deliberate room for a future second component; nothing in the codebase reads the upper half. A rebuilder should treat the width as 32 bits of real entropy.

The debug build carries two escape hatches driven by command-line switches: one dumps the serialized configuration to a file and logs each contributing file's checksum, so a mismatch can be bisected; the other takes the fingerprint verbatim from the command line so a developer can impersonate a matching client. Both are diagnostic and neither should ship enabled.

Because a pattern is matched as a *substring* of the whole path, a short pattern matches far more files than an author expects. This is the knob that decides how expensive the pass is: it reads every matched file end to end.
