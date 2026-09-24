# src/xrGame/CdkeyDecode — product-key decoding

Part of chapter 26 of [`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md#7-build-order),
alongside [`ik`](../ik/README.md) and [`gamespy`](../gamespy/README.md), with which it
shares nothing but a chapter number.

Four small files that answer one question: *is the string the player typed at install time
shaped like a product key this publisher issued?* The answer gates entry to multiplayer —
the server browser and the connect command — and nothing else. It is checked locally, with
no network access, and it is a typo detector, not an entitlement check: it cannot tell
whether a key was ever sold, only whether it is internally consistent. On every platform
but Windows the check is compiled out and always passes.

This is vendor code, taken from the matchmaking SDK with the project's own types
substituted in. Its real authority lived on a publisher's key-list server, which no longer
exists — see
[Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts).
**A rebuild may replace this whole directory with a function that returns true.** Nothing
a player can observe depends on it. It is documented because the key format is frozen and
because a rebuild that *does* want to keep the gate has to reproduce these three constants
exactly.

## Where it sits

It rests on nothing — no engine header, no allocator, no filesystem. It can be built and
tested before any other part of the game. Its one caller is the main menu, which reads the
stored key out of platform settings and asks these functions about it.

## Load-bearing ideas, named once

**A key is one string holding three fields.** Decoded, it is: a payload of one to sixteen
bytes, then an eight-byte secret, then a two-byte checksum. There is no length field —
**the number of characters the player typed is the only thing that says how long the
payload is**, and every offset in the format is computed by subtracting ten from the
decoded length. A rebuild that pads or truncates will unmask against the wrong offset.

**The text encoding is not standard base-32.** Thirty-two characters, chosen so that no
two look alike when printed on a box: the alphabet drops `I` and `O` and the digits `0`
and `1`. Eight characters carry forty bits, and within each group of eight the *last*
character supplies the highest bits while the bytes come out lowest-first. Encoding and
decoding use opposite conventions that happen to cancel. All of this is frozen by keys
already printed; a standard library's base-32 will not decode them.

**The payload is masked by the secret that sits beside it.** Exclusive-or, with the eight
secret bytes cycled if the payload is longer. This is obfuscation — the mask travels with
the thing it masks — and its only purpose is that two keys with the same payload do not
look alike. The game never reads the payload.

**The checksum is a rolling 32-bit multiply-accumulate, folded to 16 bits, then salted.**
It covers the masked payload and the secret but not itself. The salt is a per-product
identifier, so one key batch belongs to one product; the menu tries four product
identifiers in turn and accepts the first that verifies. Three constants matter and only
two have discoverable reasons: the modulus `65521` is the largest prime under 2^16, the
salt is the product identifier, and the multiplier `0x9CCF9319` is arbitrary vendor
trivia that must nonetheless be copied verbatim.

**Every failure returns the same value.** Malformed, too long, too short, bad character —
all zero, all false, no diagnostic. The player gets one dialog saying the key is invalid.

## The twins

| File | Role |
|---|---|
| [`cdkeydecode.h`](cdkeydecode.h.md) | Surface of the module: decode payload, verify checksum. |
| [`cdkeydecode.c`](cdkeydecode.c.md) | The key layout, the unmasking, the checksum, and where the result gates anything. The substantial page of the directory. |
| [`base32.h`](base32.h.md) | Surface of the text encoding. |
| [`base32.c`](base32.c.md) | The alphabet and the bit-packing, in both directions, with the two rounding rules and the reversed group order that make them frozen. |
