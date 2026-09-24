# src/utils/mp_configs_verifyer/entry_point.cpp

> The verifier's front door — three ways to feed it a dump (one file, a stream of names on standard input, or unpack-and-stop), and a fault barrier around the untrusted parse.

**Needs** — [`configs_dump_verifyer.h`](configs_dump_verifyer.h.md) · [`pch.h`](pch.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/Compression/ppmd_compressor.h`](../../xrCore/Compression/ppmd_compressor.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: it installs a process-level fault barrier around a parse of hostile input, and it sizes a decompression buffer from a length the attacker chose.

## Purpose

The composition root of the verifier. It decides three things: where the authoritative
configuration comes from, how dumps arrive, and what happens when a malicious dump crashes
the parser.

The middle one is the design. A server operator does not verify one file; they verify a
stream of them as players upload. So the tool's real mode is the **filter**: it reads file
names from standard input, one per line, forever, and prints a verdict per name. That
makes it a long-lived process a server driver pipes into, and it is why the expensive
canonical dump in [`configs_dump_verifyer.cpp`](configs_dump_verifyer.cpp.md) is built
once and reused. The single-file mode is the same thing with a batch of one, for a human
checking a report by hand.

The third is an admission. The verdict parses attacker-controlled bytes with pointer
arithmetic, so the whole call is wrapped in a process-level fault barrier: a crash inside
it is caught, reported as a failure to verify, and the loop continues. That is how a
long-lived filter survives a deliberately malformed upload. A rebuild that bounds-checks
its parse does not need the barrier — but it must still keep the property the barrier
buys: **one bad dump must not stop the stream.**

## State

```text
RECORD VerifierProcess
  mode       : ENUM { single_file, unpack, filter, help }
  file_name  : text                  # the last argument, whatever the mode
  settings   : ConfigFile            # the server's authoritative configuration
```

**Invariants**

- The authoritative configuration is loaded from a **dedicated filesystem description**,
  not the game's own — a name distinct from the one the client uses. A verifier runs
  beside a dedicated server, whose data layout is not a player's installation, and the
  separate description is what lets it point at server data without pretending to be a
  game.
- The configuration is loaded read-only and never reloaded. A server that edits its
  balance while the verifier is running will report every subsequent honest dump as a
  cheat until the verifier restarts. That is a real operational constraint and it is
  nowhere stated in the original.
- Options are recovered by **substring search over the joined arguments**, and the file
  name is always the *last* argument regardless of where the option appeared. A file whose
  name contains an option's text selects that mode.
- The decompression bound is a fixed one-megabyte ceiling on the *declared* uncompressed
  size, checked **before** allocating. It is the only thing standing between a hostile
  file and an unbounded allocation, and the ceiling is a judgement about how large an
  honest dump gets, not a property of the format.

## `main`

**Contract** — parses the mode, brings up the core layer and the authoritative
configuration for the three modes that need it, runs that mode, and tears down. Returns a
failure status when no arguments were given or the mode was not recognized; success
otherwise, including when an individual file failed to verify — a verdict is printed, not
returned, because the filter mode has many verdicts and one exit status.

```text
FUNCTION main(arguments) -> int
  print_banner()
  IF arguments.count <= 1 THEN print_usage(); RETURN failure

  file_name <- arguments[last]
  joined    <- all arguments joined with spaces

  IF joined NAMES check      THEN with_core(check_file, file_name)
  ELSE IF joined NAMES unpack THEN with_core(unpack_file, file_name)
  ELSE IF joined NAMES filter THEN with_core(run_filter)
  ELSE IF joined NAMES help   THEN print_usage()
  ELSE print_usage(); RETURN failure
  RETURN success

FUNCTION with_core(action, argument...)
  initialize_core(app_name = "mp_configs_info",
                  filesystem_description = the server's own description)
  route_log_to_standard_output()
  settings <- open_config(root "$game_config$" / "system.ltx", read_only)
  action(argument...)
  release(settings); shut_down_core()
```

**Notes**

- The log is redirected to the console rather than to a file, because the tool's whole
  output is meant to be read by whatever process is piping into it.

## `check_file` — one verdict

**Contract** — opens a dump by name, falling back to the screenshot root when the name
does not resolve directly, reads it whole, and prints either a success line or a failure
line naming the file and the reason. Reports and returns on a file it cannot open rather
than failing. Appends a zero byte past the data so the verdict's text scans can treat the
buffer as terminated text.

```text
FUNCTION check_file(name)
  reader <- open(name)
  IF reader IS none
    rescan(root "$screenshots$")          # the client uploads dumps alongside screenshots
    reader <- open(root "$screenshots$" / name)
    IF reader IS none THEN report("cannot open " + name); RETURN

  data <- read_all(reader) + one zero byte
  WITHIN FAULT BARRIER
    IF verify(data, OUT reason) THEN report("GOOD")
    ELSE report("CHEATER (" + name + "): " + reason)
  ON FAULT
    report("FATAL ERROR: failed to verify data")
```

**Notes**

- Dumps arrive through the **screenshot** root because that is the channel the game
  already had for a client sending a file to a server operator. It is an accident of
  plumbing, not a decision about dumps, and the rescan before the fallback exists because
  the root's directory listing is cached and a freshly uploaded file would otherwise not
  be found.
- A fresh verifier is constructed per file, which rebuilds the canonical dump every time.
  In the filter mode — the one that matters — that is a real and unnecessary cost: the
  canonical dump depends only on the configuration, which does not change. A rebuild
  should hoist it out of the loop.

## `unpack_file` — recovering a dump for a human

**Contract** — reads a compressed dump, decompresses it, and writes the plain text beside
it with the extension changed. Refuses a file whose declared uncompressed size exceeds the
ceiling, and refuses one whose actual decompressed size does not match what it declared.
Prints a reason and returns on any failure. Does not verify anything.

```text
FUNCTION unpack_file(name)
  reader <- open(name) OR open(root "$screenshots$" / name)
  IF reader IS none THEN report("cannot open " + name); RETURN

  declared <- reader.read_int()            # first word of the file
  payload  <- read_rest(reader)
  IF declared > one_megabyte THEN report("bad archive"); RETURN

  plain <- decompress(payload, into a buffer of exactly `declared` bytes)
  IF length(plain) != declared THEN report("bad archive: size mismatch"); RETURN

  write_all(open_for_write(name with ".ltx" in place of ".cltx"), plain)
```

**Invariants**

- **The compressed file's first word is the uncompressed length.** The compressor's own
  stream does not carry it, and the decompressor needs the output buffer sized in advance,
  so the packer prepends it. That word is the format, such as it is: four bytes of length
  and then the compressed stream.
- Checking the declared length against the ceiling **before** allocating, and the produced
  length against the declared one **after**, are two separate guards against two separate
  attacks — an allocation bomb and a stream that expands past its buffer. Both are needed.
- The output name is derived by replacing the compressed extension if present and
  appending otherwise, so a file with an unexpected name still produces a distinct output
  rather than overwriting its input.

## `run_configs_verifyer_server` — the filter

**Contract** — reads whitespace-separated file names from standard input and checks each,
until the input ends. Never returns a verdict as a status; every result is a printed line.
Blocks on input.

```text
FUNCTION run_filter()
  WHILE read_token(standard_input) SUCCEEDS
    check_file(token)
```

**Notes**

- Names are read as whitespace-separated tokens into a fixed-width path buffer, so a name
  containing a space is split and a name longer than the buffer overruns it. Both are
  defects, and both are invisible in practice because the driver generates the names. A
  rebuild reads a line at a time into a growable buffer.
