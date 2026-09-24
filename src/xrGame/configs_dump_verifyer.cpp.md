# src/xrGame/configs_dump_verifyer.cpp

> The server's check: rebuild what a client's configuration dump *should* have been, compare, and name the first thing that differs.

**Needs** — [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`configs_common.h`](configs_common.h.md) · [`configs_dumper.h`](configs_dumper.h.md) · [`mp_config_sections.h`](mp_config_sections.h.md) · [`xrCore/Crypto/xr_dsa_verifyer.h`](../xrCore/Crypto/xr_dsa_verifyer.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md)
**Used by** — [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md)
**Tier floor** — T1: it reassembles a byte-exact signed region by pointer surgery on a received buffer

## Purpose

The server side of the scheme whose client side is
[`configs_dumper.cpp`](configs_dumper.cpp.md). Given a client's dump, it answers two
questions with one comparison: is this dump intact, and does it describe the same
configuration the server is running?

The method is to build the server's *own* dump of the same material, hash it, and compare
that hash against the one the client's signature vouches for. A single hash comparison
covers the entire configuration, which is why the dump has to be byte-reproducible on both
sides. When they differ, a second, slower pass walks the received document to find a
difference a human can read, because "the hashes differ" is useless to a server operator.

## State

```text
RECORD configs_verifyer
  original_body     : byte buffer     # the server's own configuration sections, dumped once
  original_body_end : int             # where they end; everything after is per-client
  verifyer          : the verification context, built from the shared domain and public key
  original_config   : the configuration-section enumerator
  original_ap       : the live-parameter reference
```

**Invariants**

- The server's own section dump is produced **once**, at construction, and reused for every
  client. Only the client-specific tail — the active parameters and the identity string — is
  rewritten per verification, by seeking back to the remembered end. This is what makes the
  check cheap enough to run on every joining client.
- The buffer is therefore not reentrant: two verifications cannot run at once.
- The server's section dump must be produced by *exactly* the same enumerator the client
  used, in the same order. Everything rests on that.

## `verify`

**Contract** — takes the received dump and a buffer for a human-readable difference. Returns
whether the client's configuration matches. On failure the difference buffer holds either a
named mismatch or one of three fixed reasons: the dump is not a dump, its signature does not
check, or the contents differ in some way the second pass could not localize.

```text
FUNCTION verify(data, size, OUT diff) -> bool
  IF the literal "[config_dump_info]" does not occur in data THEN
    diff = "invalid dump"; RETURN false          # not a dump at all

  received = parse data as a configuration image

  # rebuild the server's expectation of the client's ACTIVE parameters:
  # the dump lists, under a known section, which parameter sections it dumped,
  # numbered from one. For each, take the SERVER's version of that section.
  expected_params = empty
  FOR index = 1, 2, 3, ... WHILE received has an entry for index
    name = received.active_params[index]
    expected_params.active_params[index] = name
    IF expected_params has no section `name` THEN
      copy the server's own `name` section into expected_params

  original_body.position = original_body_end
  append expected_params to original_body

  IF the identity section lacks any of name, digest, date, signature THEN
    diff = "invalid dump"; RETURN false

  identity = received name THEN received digest THEN received date
  append identity as a zero-terminated string to original_body
  expected_hash = digest of original_body

  IF NOT verify_signature(data, size, OUT claimed_hash) THEN
    diff = "invalid digital sign"; RETURN false

  IF claimed_hash != expected_hash THEN
    find_difference(received, expected_params, OUT diff)
    RETURN false

  RETURN true
```

**Invariants** — the identity string is assembled from the *received* fields and appended to
the *server's* body. So the hash covers the server's configuration plus the client's
identity, and the signature covers the client's configuration plus the same identity. The
two agree exactly when the configurations agree. The order of the three identity fields and
the terminator are frozen protocol shared with the dumper.

**Notes** — the cheap "is this a dump" test is a literal substring search for the identity
section's header. It runs before any parsing, so a random or hostile buffer is rejected
without going through the configuration parser.

## signature verification

**Contract** — reconstructs the exact bytes the client signed and checks the signature over
them, yielding the digest the client vouched for.

The client transmitted the identity *section* where it had signed the identity *string*, so
the reconstruction is: find the identity section, cut the buffer there, and append the
identity string built from that section's fields.

```text
FUNCTION verify_signature(data, size, OUT hash) -> bool
  at = the LAST occurrence of the identity section's name in data
  IF none THEN RETURN false
  at = at - 1                        # step back onto the section's opening bracket

  info = parse the text from `at` onward as a configuration image
  IF it lacks name, digest, date or signature THEN RETURN false

  truncate data at `at`
  append (received name THEN digest THEN date) as a zero-terminated string
  hash = verify(data, its new length, against the signature from `info`)
  RETURN whether the signature checked
```

**Invariants** — the search runs **backwards from the end**, because the identity section is
written last and a section of that name could in principle appear earlier in the dumped
configuration. Stepping back one byte to land on the opening bracket assumes the section
header is written in the canonical form with no leading whitespace, which the writer
guarantees.

**Notes** — this rewrites the caller's buffer in place: it plants a terminator and appends
past the end of the received data. The buffer must therefore have slack after it, and it is
unusable afterwards. A rebuild should copy rather than overwrite; the saving is not worth
the coupling.

## finding a readable difference

**Contract** — the slow path, run only after a hash mismatch. Walks every section of the
received document except the identity section and the active-parameter index, and reports the
**first** key that disagrees with the server or is not present on the server at all. The
message names the section, the key, the value the client claimed and the value the server
expects — enough for an operator to say "this player edited the damage on this weapon".

```text
FUNCTION find_difference(received, expected_params, OUT diff) -> text
  FOR EACH section IN received
    IF section is the identity section or the active-parameter index THEN CONTINUE
    is_live = section name begins with "ap_"        # a live-parameter section
    FOR EACH (key, claimed) IN section
      IF is_live AND expected_params has (section, key) THEN
        IF claimed != expected value THEN RETURN "<section>::<key> = <claimed>, right = <expected>"
        CONTINUE
      IF the server's configuration has no (section, key) THEN
        RETURN "line <section>::<key> not found"
      IF claimed != the server's value THEN
        RETURN "<section>::<key> = <claimed>, right = <expected>"
  RETURN "unknown diff or corrupted config dump"
```

**Invariants** — the `ap_` prefix is what separates a *live parameter* section (what the
object is running with) from an ordinary configuration section (what the file declares). Live
sections are compared against the reconstructed expectation; everything else against the
server's own configuration. The prefix is frozen protocol between this file and the
live-parameter dumper.

**Notes** — a live-parameter section whose key the expectation does not contain falls through
to being compared against the *static* configuration. That is a deliberate fallback: a
parameter that has not been modified reads the same either way.

**Notes** — the walk stops at the first difference, so an operator sees one at a time. That is
a usability choice, not a limitation of the data.

**Notes** — the final fallback message is reachable, and its existence is an admission: the
hashes can differ while every compared key agrees, because the hash covers ordering and
formatting that the key-by-key walk does not. Section *order* differing is enough. A rebuild
that canonicalizes the dump before hashing removes this whole class of unexplainable
rejection.
