# src/xrGame/CdkeyDecode/cdkeydecode.c

> A product key is three concatenated fields — payload, secret, checksum — encoded as one
> readable string; this recovers the payload by unmasking it with the secret, and checks
> the string's own consistency without contacting anybody.

**Needs** — [`base32.h`](base32.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — reached through its declarations in [`cdkeydecode.h`](cdkeydecode.h.md); callers name that, not this file.
**Tier floor** — T2. The checksum relies on 32-bit unsigned multiply wrapping, and the
stored checksum is read as a 16-bit little-endian field; both are frozen by keys printed
in 2007. Nothing else here constrains the tier.

## Purpose

Decodes and sanity-checks the product key the player typed at install time. Its only real
job in this engine is to answer *"does this key look plausible"* before the multiplayer
menu lets the player reach the (now dead) matchmaking service. The key's payload is never
used for anything the player can observe.

This is client-side validation in the literal sense: it proves the string is
self-consistent, and nothing else. A key that passes may still never have been sold; a
key that was sold and then revoked still passes. Real authority lived on the publisher's
server, which is gone — see the matchmaking seam.

## State

Stateless. The layout of a decoded key, however, is frozen and is the file's real content:

```text
RECORD DecodedKey               # the byte string a key decodes to, in this order
  payload : bytes               # 1..16 bytes; masked, see below
  secret  : bytes (8)           # the mask, and the thing the publisher's key batch fixes
  check   : int (16-bit, little-endian)
```

Invariants that are enforced only by arithmetic, nowhere by a declaration:

- Total decoded length is `len(payload) + 8 + 2`, so `len(payload)` is whatever is left
  over and must be at least 1. A key that decodes to 10 bytes or fewer carries no payload
  and is rejected.
- `len(payload)` is capped at 16 by the buffer the caller sizes, not by any field in the
  key. There is no length field: **the key's character count is the only thing that says
  how long the payload is.** A rebuild must derive it the same way or it will unmask
  against the wrong offset.
- The longest key the decoder will accept is the encoding of 26 bytes — 42 characters,
  before separators.

## `DecodeKeyData`

**Contract** — takes the key as typed (separators and case are tolerated) and a
caller-supplied output buffer of at least 16 bytes; writes the recovered payload and
returns its length in bytes. Returns 0 for *every* failure — too long, an out-of-alphabet
character, a key too short to contain payload — with no distinction between them and no
diagnostic. Does not allocate, does not block.

```text
FUNCTION decode_key_data(key : text) -> optional<bytes>
  clean <- strip separators, upper-case          # see CleanForBase32
  IF clean did not fit in the maximum key length
    RETURN none
  raw <- decode_base32(clean)
  IF raw failed OR length(raw) <= 10             # 8 secret + 2 check leaves nothing
    RETURN none

  n <- length(raw) - 10                          # payload length, by subtraction
  payload <- n bytes
  FOR i FROM 0 TO n - 1
    # The mask is the secret, cycled. For payloads of 8 bytes or less this is a plain
    # byte-for-byte exclusive-or; longer payloads reuse the same 8 mask bytes.
    payload[i] <- raw[i] XOR raw[n + (i MOD 8)]
  RETURN payload
```

**Notes** — the masking is obfuscation, not encryption: the secret sits in the same string
as the thing it masks, so anyone holding the key holds the mask. What it buys is that two
keys carrying the same payload do not look alike, which is enough to stop a player reading
a batch number off the box.

Reusing the mask across a payload longer than the secret is a classic repeating-key
weakness. It does not matter here, and a rebuild should reproduce it rather than improve
it, because the keys are already printed.

Nothing in this engine consumes the returned payload. It exists for the publisher's own
tooling — batch identification and territory coding — and the function survives because
the vendor header declared it.

## `VerifyClientCheck`

**Contract** — takes the key as typed and a 16-bit salt, and answers whether the key's
stored checksum matches one recomputed over the key's own bytes. Returns true or false;
a key that cannot be decoded at all returns false. No I/O, no allocation.

```text
FUNCTION verify_client_check(key : text, salt : int (16-bit)) -> bool
  clean <- strip separators, upper-case
  IF clean did not fit OR decode fails
    RETURN false
  raw <- decode_base32(clean)
  n <- length(raw) - 10                          # payload length

  # Checksum covers the MASKED payload and the secret — everything but the stored
  # check field itself.
  accumulator <- 0 as int (32-bit, wraps)
  FOR EACH b IN first (n + 8) bytes OF raw
    accumulator <- accumulator * 0x9CCF9319 + b

  expected <- (accumulator MOD 65521) XOR salt   # taken as a 16-bit value
  stored   <- raw[n + 8 .. n + 9] read as int (16-bit, little-endian)
  RETURN expected == stored
```

**Invariants** — the multiply must wrap at 32 bits and the modulus must be applied to the
*truncated 16-bit* accumulator, in that order; both choices change the answer and both
are frozen by the shipped keys.

**Notes** — three constants, and what is recoverable about each:

- `0x9CCF9319` is the rolling multiplier. It is an odd 32-bit constant, which is all the
  arithmetic requires; **why this particular value was chosen is not discoverable from the
  source** and it is almost certainly arbitrary vendor trivia. It must be reproduced
  exactly.
- `65521` is the largest prime below 2^16 — the same modulus Adler-32 uses. It spreads the
  accumulator across the full 16-bit field without a bias toward low values.
- The salt is exclusive-ored in last, so the same key checks out under one salt and fails
  under another. It is how one key batch is made to belong to one product. The caller
  supplies it as a *game identifier*, and tries four in turn, because one installation may
  hold a key issued for any of several product generations — the first that verifies wins.

Note the asymmetry with the decoder: `VerifyClientCheck` does **not** reject a key that
decodes to ten bytes or fewer. With a payload length computed as zero or negative it
reads the stored check from an offset it derived from a nonsensical length. The decoder's
guard is the only thing that makes the pair safe, and callers reach this function first.
A rebuild should apply the decoder's length guard here too.

## Where the result is used

Two menu paths and nothing else, and only on Windows — elsewhere the check is compiled to
an unconditional pass, which is the honest statement of how much it is worth:

- Before the multiplayer server browser opens, and before the console command that starts
  a client connection runs. A failure raises the "invalid key" error dialog and the
  action is abandoned.
- The same predicate is exported to the script layer, so a mod's menu can gate on it.

A key read back as empty is treated as valid outside demo builds, so a normal installation
with no key recorded is not blocked. The gate exists to keep unlicensed clients off the
publisher's matchmaking service; with that service gone, a rebuild may implement this
module as a constant `true` and lose nothing a player can see.
