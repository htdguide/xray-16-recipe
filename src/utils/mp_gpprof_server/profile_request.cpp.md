# src/utils/mp_gpprof_server/profile_request.cpp

> Turns a request path into a player name, and a profile into the response that answers it.

**Needs** — [`profile_request.h`](profile_request.h.md) · [`profile_printer.h`](profile_printer.h.md) · [`profile_data_types.h`](profile_data_types.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)

**Used by** — reached through its declarations in [`profile_request.h`](profile_request.h.md); callers name that, not this file.

**Tier floor** — T2: text handling plus a connection whose release is ordered.

## Purpose

The tool's whole public interface is one path shape and one response shape, and both are
here. A request for `<root>/<player name>` is answered with that player's profile as plain
text; anything else is answered "not found".

## State

```text
RECORD PendingRequest
  connection   : the accepted request, owned until completion
  profile_name : text        # decoded, bounded by the name buffer's width
```

**Invariants**

- **Completing a request is what releases the connection**, and it happens exactly once.
  Both completions set a status and then finish the request; nothing else may touch it
  afterwards.
- The name is copied out of the path at acceptance, so the request object does not depend
  on the path buffer surviving.

## `extract_username`

**Contract** — given a request path and the configured root path, returns the player name
that follows the root, with its escaping undone. Returns empty text when the path does not
begin with the root or names nothing after it, and the caller treats empty as "not found".
Never fails; never allocates.

```text
FUNCTION extract_username(path, root) -> text
  # Take the single path segment that follows the root.
  raw <- the first whitespace-free run after root + "/" in path
  IF raw IS empty THEN RETURN ""
  RETURN percent_decode(raw), truncated to the name buffer's width
```

**Invariants**

- The match is anchored at the root, so a request outside it yields nothing and is refused.
  The root is configurable precisely so the service can be mounted under whatever prefix
  the web server in front of it uses.
- Only one segment is taken. A path with further segments yields the first, so
  `<root>/name/anything` answers about `name`.

### Percent decoding

**Contract** — replaces each escape — an introducer followed by two hexadecimal digits —
with the byte it denotes, copying every other character through. Stops early on a truncated
escape at the end of the input. Never lengthens the text, so it decodes in place into a
buffer of the source's size.

**Notes**

- **Player names are not restricted to plain characters**, which is the entire reason this
  exists: the service's accounts allow spaces and punctuation, and a name reaches the tool
  escaped. The decode is byte-oriented and encoding-blind, which is correct here because
  what the name is eventually compared against is also bytes.
- The decode does not validate that the two characters after an introducer are actually
  hexadecimal; a malformed escape decodes to whatever the conversion yields, usually zero,
  which truncates the name. Harmless — it produces a lookup miss — but a rebuild should
  reject rather than mangle.
- **A name containing a quote is not filtered here**, and it must be: the name goes into a
  filter expression sent to the service. That filtering happens later, in
  [`gamespy_sake.cpp`](gamespy_sake.cpp.md), which is a long way from where the name
  enters. A rebuild should reject the character at the door.

## `complete_success`

**Contract** — writes the response body and closes the request with a success status. The
body is a blank line, the player's name, the rendered profile, and a trailing line break.
Releases the connection.

```text
FUNCTION complete_success(profile)
  write(output, line_break)                 # ends the (empty) response header block
  write(output, profile_name, line_break)
  write(output, render(profile))            # profile_printer.h
  write(output, line_break)
  set_status(success)
  finish()
```

**Invariants**

- **The leading blank line is structural**: the protocol in front of this tool expects the
  response headers, then a blank line, then the body, and this tool emits no headers at
  all — so the body begins with the separator. A consumer parsing the response must skip
  it.
- The player's name is echoed as the first body line so a consumer batching several
  lookups can tell the answers apart. It is echoed as *requested*, not as the service
  spelled it.

## `complete_failed`

**Contract** — closes the request with a not-found status and no body at all. Used for a
path that names nobody, and for a name the service had no record of — the two are
deliberately indistinguishable to the caller.

**Notes**

- The status is set on the *output* channel in both completions, which is how the
  surrounding protocol carries it. That is a property of the gateway interface, not a
  decision of this tool.
