# src/xrGame/secure_messaging.cpp

> Obfuscates network message payloads with a seed-derived keystream chained against the previous word, and returns a plaintext checksum as the tamper check.

**Needs** — [`secure_messaging.h`](secure_messaging.h.md) · [`xrCore/_random.h`](../xrCore/_random.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: in-place word-width transform over a byte buffer, including a partial trailing word

## Purpose

Multiplayer clients and the server exchange a seed; each derives the same key from it and
scrambles message payloads with it. This is **not** cryptography and must not be described
as such — it is an anti-casual-tampering measure with a rolling checksum, of the kind that
raises the cost of a naive packet editor and nothing more. A rebuild that wants real
confidentiality should replace the whole file with an authenticated cipher; a rebuild that
wants to interoperate with the original's clients must reproduce it exactly, because the
transform is part of the wire format.

## State

```text
RECORD Key
  length : int          # invariant: 16 <= length <= 32, drawn from the seed
  words  : list<int (32-bit)>   # `length` of them used, capacity always 32

RECORD SeedGenerator
  random : random stream        # seeded once from the low 32 bits of the cycle counter
```

**Invariants** — the key is a pure function of the seed. Both ends derive it
independently; the seed is what travels, never the key. Because the key length itself is
drawn from the seeded stream *before* the words, two peers that disagree about the
generator's algorithm diverge on the very first word — there is no length field to
cross-check.

## `seed_generator`

**Contract** — constructed once per peer, seeded from the low half of the processor cycle
counter at construction time. Each call yields the next value of the stream. Not
thread-safe and not copyable: a copy would replay the same seeds, which would let two
message streams share a key.

**Notes** — seeding from a cycle counter is the weak point and is worth naming. It gives
an attacker who can estimate uptime a small search space. Keep the interface; replace the
source with a real entropy source in a rebuild.

## `generate_key`

**Contract** — deterministic. Draws the key length from a fresh random stream seeded with
the given seed, then fills that many words:

```text
FUNCTION generate_key(seed) -> Key
  stream = random stream seeded with seed
  key.length = stream.next_in_range(16, 32)
  FOR i IN 0 .. key.length - 1
    key.words[i] = (stream.next() SHIFTED LEFT 17)
                 OR (stream.next() SHIFTED LEFT 2)
                 OR (stream.next() AND 3)
  RETURN key
```

**Notes** — three draws per word, shifted and merged, because the underlying stream's
output is not full-width: the composition is what fills all 32 bits. The shift amounts
(17, 2, and a two-bit tail) are not derivable from anything stated in the source; they are
tuned to the particular generator's usable bit range and a rebuild must copy them verbatim
to stay compatible.

## `encrypt` and `decrypt`

**Contract** — both call one in-place transform over the buffer, differing only in which
side of the exclusive-or supplies the chaining word and which side is summed into the
returned checksum. Both return the checksum of the **plaintext** — which is the point: the
receiver decrypts, gets a checksum, and compares it with the checksum the sender computed
while encrypting. A payload altered in flight produces a different sum.

```text
FUNCTION transform(buffer, size, key, direction) -> int (32-bit, wraps)
  previous = -1                      # all bits set: the initialization vector
  key_index = 0
  checksum = 0

  FOR EACH word IN whole 32-bit words of buffer
    raw = word
    word = (raw XOR key.words[key_index]) XOR previous
    IF direction IS encrypt
      previous = raw                 # chain on the plaintext
      checksum = checksum + raw
    ELSE
      previous = word                # chain on the recovered plaintext
      checksum = checksum + word
    key_index = (key_index + 1) MOD key.length

  # the trailing partial word, if any
  rest = size MOD 4
  IF rest > 0
    raw = the remaining `rest` bytes, zero-extended to a word
    out = (raw XOR key.words[key_index]) XOR previous
    write back the low `rest` bytes of out
    IF direction IS encrypt
      checksum = checksum + raw
    ELSE
      checksum = checksum + (out masked to the low `rest` bytes)
  RETURN checksum
```

**Invariants** —

- The chain word is always the **plaintext** of the preceding word, on both sides. That is
  what makes decryption the same loop with one assignment moved: the encryptor remembers
  what it read, the decryptor remembers what it produced, and both hold the same value.
- The key index wraps at the key's *derived* length, not its capacity. A key of 16 words
  repeats every 16; one of 32 every 32.
- The initialization vector is all-ones, not zero, so a buffer of zeros does not encrypt
  to the raw keystream.
- The trailing partial word is transformed but **does not advance the chain or the key
  index**, because it is the last thing in the buffer. It is read and written through the
  partial-length copy, so the bytes past the end of the buffer are never touched.
- Checksum addition wraps at 32 bits, deliberately.

**Notes** — the trailing word's checksum contribution is asymmetric and this is the file's
one genuine subtlety. Encrypting sums the zero-extended raw bytes; decrypting sums the
*masked* output, because the high bytes of the recovered word are garbage from the
exclusive-or and would otherwise poison the sum. The mask is built by shifting an all-ones
word right by eight bits per missing byte. A rebuild that gets this wrong sees checksums
agree on every message whose length is a multiple of four and disagree on the rest.

The direction enumeration carries a comment forbidding new values. That is because the
transform's two branches are the *only* two chaining rules that invert each other; a third
direction would be meaningless, not merely unimplemented.
