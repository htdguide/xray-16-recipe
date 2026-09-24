# src/utils/mp_configs_verifyer/configs_dump_verifyer.cpp

> Decides whether an uploaded configuration dump was produced by a client running the same balance data as the server, by rebuilding the dump locally and comparing digests through the client's signature.

**Needs** — [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`configs_common.h`](configs_common.h.md) · [`mp_config_sections.h`](mp_config_sections.h.md) · [`pch.h`](pch.h.md) · [`xrCore/Crypto/xr_dsa_verifyer.h`](../../xrCore/Crypto/xr_dsa_verifyer.h.md) · [`xrCore/Crypto/xr_sha.h`](../../xrCore/Crypto/xr_sha.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [Data: Configuration](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: it locates a structure inside an untrusted byte buffer by scanning for a marker and then truncates that buffer in place, and it feeds an exactly-sized digest to a signature primitive.

## Purpose

This is the whole anti-cheat, in one idea: **a client proves it is running the server's
balance data by signing a digest of that data, and the server checks the signature against
a digest it computed itself.** If the two digests agree, the client's configuration is
byte-identical to the server's over the protected set. If they do not, the signature still
tells the server what digest the client *did* sign, and the server then walks the client's
own dump line by line against its configuration to say which value differs — which turns
"cheater" into "cheater, and here is the line".

Everything else on this page is the machinery that makes that comparison well-defined: the
dump's layout, what exactly is hashed, and where the boundary between signed and unsigned
bytes falls.

## State

```text
RECORD DumpVerdict
  canonical      : bytes     # the server's own rendering of the protected set
  canonical_end  : int       # where the protected set ends and the per-dump suffix begins
  signature_check: SignatureChecker    # bound to the frozen parameters
```

**Invariants**

- `canonical` is built **once**, at construction, by serializing every section of the
  protected set in order — see
  [`mp_config_sections.cpp`](mp_config_sections.cpp.md). It is the expensive part and it
  is reused for every dump checked in a run. A server checking thousands of uploads pays
  for it once.
- `canonical_end` is the length of that static prefix. Every dump appends its own dynamic
  sections and its own identity suffix after it, so the buffer is rewound to this mark
  before each verdict. Forgetting the rewind silently makes each verdict depend on the
  previous one.
- **The digest covers the protected set, then the dynamic sections, then the identity
  triple, in that order, with no separator.** The order is the protocol; there is nothing
  in the bytes that would let a reader recover the boundaries. Both ends agree by
  construction or not at all.
- **The identity triple is player name, player digest, creation date, concatenated with no
  separator and terminated by a zero byte.** It is what binds a dump to a player and a
  moment: without it, one valid dump could be replayed by anyone forever. With no
  separator, a name ending in digits and a digest starting with digits are
  indistinguishable — a genuine ambiguity nobody exploited because all three come from the
  same client that is also signing.

## The dump's layout

```text
RECORD ConfigDump                    # what the client uploads, after decompression
  ...protected sections...           # the same sections the server will render itself
  [active_params_section]            # names of the dynamic sections, keyed 1, 2, 3, ...
  ...dynamic sections...             # each named "ap_<something>", contents as the client saw them
  [config_dump_info]                 # the identity suffix, always last
    player_name    : text
    player_digest  : text
    creation_date  : text
    digital_sign   : text            # over everything above, plus the triple, minus itself
```

**Invariants**

- The identity section is **last**, and that is what makes the scheme work at all: the
  signature cannot cover itself, so the signed region is *everything before the identity
  section*, plus the three identity values appended after it. The verifier reconstructs
  that region by truncating the buffer at the identity section's start and appending the
  triple there.
- The dynamic section names are listed under consecutive numeric keys starting at one, and
  the list ends at the first key that is absent. A gap truncates the list silently.
- A dynamic section's name begins with a three-character prefix that marks it as dynamic.
  That prefix is the only thing distinguishing "check this against the live object's
  expected parameters" from "check this against the shipped configuration".

## `verify` — the verdict

**Contract** — takes the decompressed dump as a mutable byte buffer and its length, returns
whether it verifies, and on failure writes a short human-readable reason into the caller's
buffer. **Mutates the input**: it truncates the buffer at the identity section and appends
to it, so the caller must not reuse it. Fails, with distinct reasons, when the buffer does
not look like a dump at all, when the identity section is incomplete, when the signature
does not verify, and when it verifies but the digests differ. Not thread-safe: the
canonical buffer is shared state. Does not allocate beyond the scratch configuration
objects.

```text
FUNCTION verify(dump : bytes, OUT reason : text) -> bool
  IF dump DOES NOT CONTAIN "config_dump_info"
    reason <- "invalid data"; RETURN false

  received <- parse_config(dump)

  # Rebuild the dynamic half from the *names* the dump supplies and the server's own values.
  active <- empty writable config
  index  <- 1
  WHILE received HAS line active_params_section / text(index)
    name <- value_of(received, active_params_section, text(index))
    set(active, active_params_section, text(index), name)   # the name list is signed too
    load_active_section(name, active)                       # the server's values for it
    index <- index + 1

  # Assemble what the client *should* have hashed.
  canonical.truncate_to(canonical_end)
  write_config(active, canonical)

  FOR EACH key IN [player_name, player_digest, creation_date, digital_sign]
    IF received HAS NO line config_dump_info / key
      reason <- "invalid dump"; RETURN false

  canonical.append_zero_terminated(
      value(player_name) + value(player_digest) + value(creation_date))

  expected <- digest(canonical)

  signed_digest <- verify_signature(dump)        # also truncates and rebuilds `dump`
  IF signed_digest IS none
    reason <- "invalid digital sign"; RETURN false

  IF signed_digest != expected
    reason <- first_difference(received, active)
    RETURN false
  RETURN true
```

**Notes**

- The dynamic half is rebuilt from the server's configuration, **not** copied from the
  dump. So a client that alters a dynamic value produces a dump whose signed digest covers
  its altered text while the server's reconstruction covers the honest text, and the
  digests diverge. The dump's own copy of those values is then only used to *explain* the
  failure.
- The identity section is checked for completeness twice — once here and once inside the
  signature check — because each reads the dump through a different window. Harmless
  duplication; a rebuild parses once.

## `verify_dsign` — recovering the signed digest

**Contract** — finds the identity section in the raw buffer, reads the four identity
values out of it, reconstructs the exact byte region the client signed, and asks the
signature checker to verify. On success returns the digest that was signed; on any
failure returns nothing. **Mutates the buffer**: it writes a terminator at the identity
section's start and then appends the identity triple there.

```text
FUNCTION verify_signature(dump : bytes) -> optional<digest>
  start <- LAST occurrence of "config_dump_info" in dump
  IF start IS none THEN RETURN none
  start <- start - 1                     # step back onto the section's opening bracket

  info <- parse_config(bytes from start to end)
  IF info lacks any of the four identity keys THEN RETURN none

  # The signed region is everything before the identity section, plus the triple.
  dump.truncate_at(start)
  dump.append_zero_terminated(
      value(player_name) + value(player_digest) + value(creation_date))

  RETURN check_signature(dump, value(digital_sign))   # returns the digest that was signed
```

**Invariants**

- The search is **backwards from the end**. The marker text could legitimately appear
  inside an earlier value, and only the last occurrence is the real section. Searching
  forwards finds the decoy.
- Stepping back one byte from the marker is what includes the section's opening bracket in
  the truncation, so the signed region ends just before `[`, not just before the name. One
  byte either way changes the digest.
- The signature check is the only place a **failure to verify** and a **mismatch** are
  distinguished. A forged or corrupt signature is a different diagnosis from an honest
  signature over different data, and the operator wants to know which.

**Notes**

- The buffer is required to be at least as long as the marker text, and nothing here
  checks that on a short or empty file. The caller guards it by refusing an empty file;
  the entry point additionally runs the whole verdict under a fault barrier, which is the
  real admission that this code parses hostile input with pointer arithmetic. A rebuild
  should bounds-check and drop the barrier.

## `get_diff` and `get_section_diff` — naming the difference

**Contract** — walks every section of the received dump except the identity section and
the dynamic-name list, and returns the first line that disagrees with the authority for
that section. Returns a fixed "unknown difference or corrupted dump" text when nothing
disagrees — which happens when the dump was altered somewhere the comparison does not
look. Never fails; the result is always a message.

```text
FUNCTION first_difference(received, active) -> text
  FOR EACH section IN received.sections
    IF section.name IS config_dump_info OR active_params_section THEN CONTINUE
    dynamic <- section.name STARTS WITH the dynamic prefix

    FOR EACH item IN section.items
      IF dynamic AND active HAS line section.name / item.key
        IF item.value != value_of(active, section.name, item.key)
          RETURN qualified(section, key) + " = <theirs>, right = <ours>"
        CONTINUE
      IF settings HAS NO line section.name / item.key
        RETURN "line " + qualified(section, key) + " not found"
      IF item.value != value_of(settings, section.name, item.key)
        RETURN qualified(section, key) + " = <theirs>, right = <ours>"
  RETURN "unknown diff or corrupted config dump"
```

**Invariants**

- A dynamic section falls back to the shipped configuration when the reconstructed dynamic
  half has no line for that key. That is deliberate: a dynamic section is a *view* of a
  live object whose parameters mostly come from configuration, so most of its keys are
  answerable from configuration alone.
- A key present in the dump and absent from the server is reported as "not found" rather
  than as a mismatch, because it means the client invented a line — a different and more
  interesting kind of wrong.
- `qualified` joins a section name and a key with a two-colon separator — that
  spelling is the message format, and log scrapers were written against it.
- The message is truncated to a fixed width. It is a log line, not a report; the operator
  who wants the whole picture unpacks the dump and diffs it.

**Notes**

- This runs **only after** the digest comparison has already failed, so it is allowed to
  be slow and allowed to be incomplete. It explains a failure; it never decides one. The
  "unknown difference" answer is the honest admission that the digest covers more than
  this walk does — the rendered whitespace, the section order, the dynamic name list —
  and any of those differing produces a mismatch this walk cannot name.
