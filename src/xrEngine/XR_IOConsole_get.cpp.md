# src/xrEngine/XR_IOConsole_get.cpp

> Reading a console variable's value back out with its type and its bounds, for code that wants the setting rather than the command.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a name lookup plus a type test per read

## Purpose

The console registry is also the engine's settings store: renderer quality, sound volumes,
input sensitivity and a few hundred other values live there and nowhere else. Code that
needs one of those reads it here, by name, with the bounds the command declared — because
the bounds are the authored range and a consumer that wants to present a slider needs them.

The file exists because a typed read is not a single lookup: the registry is heterogeneous,
so each read must find the command *and* establish that it is the kind that has the
requested type.

## State

`Stateless.`

## The reads

**Contract** — every read takes a command name and returns a value plus, where it applies,
the bounds that command declared. A name that does not exist, or a command of the wrong
kind, yields the neutral value rather than failing — the console is queried from paths where
a missing setting must not stop the frame.

```text
FUNCTION get_command(name) -> optional<Command>

FUNCTION get_bool(name) -> bool
  # true for a set flag or a non-zero integer; false for anything else, including a miss
  cmd = get_command(name)
  IF cmd IS a flag command       RETURN cmd.value != 0
  IF cmd IS an integer command   RETURN cmd.value != 0
  RETURN false

FUNCTION get_integer(name) -> (value, min, max)
  # defaults (0, 0, 1): a missing setting reads as a disabled boolean
  cmd = get_command(name)
  IF cmd IS an integer command   RETURN (cmd.value, cmd.bounds)
  IF cmd IS a flag command       RETURN (cmd.value ? 1 : 0, 0, 1)
  RETURN (0, 0, 1)

FUNCTION get_float(name) -> (value, min, max)
  IF cmd IS a float command      RETURN (cmd.value, cmd.bounds)
  RETURN (0, 0, 0)

FUNCTION get_string(name) -> optional<text>
  # a command's *rendered status*, not an internal value: whatever it prints for itself
  cmd = get_command(name)
  IF cmd DOES NOT EXIST          RETURN none
  RETURN cmd.status_text()

FUNCTION get_token(name) -> optional<text>        # the same as get_string
FUNCTION get_token_table(name) -> optional<TokenTable>
  # the full set of names a token command accepts, for building a menu of them

FUNCTION get_vector(name) -> vector3               # zero when absent
FUNCTION get_vector_ptr(name) -> optional<ref>     # writable; see note
```

**Notes** — three decisions here matter to a rebuild.

**A flag reads as an integer and an integer reads as a boolean.** Both directions are
supported because the same setting is read both ways by different call sites, and the
console's own types do not distinguish a one-bit integer from a flag. The default bounds for
a missing integer are 0..1 for exactly that reason.

**`get_string` returns the command's *status text*, not a stored string.** Every command can
render itself for display — that is what typing its name with no argument prints — and
string reads reuse that. The consequence is that the returned text is formatted for a human,
and code that parses it is parsing a display format. It is also returned in a single shared
buffer that the next call overwrites, so a caller must copy it before the next read.

**`get_vector_ptr` hands out a writable reference into the command.** That is a deliberate
escape hatch for a handful of debug call sites that steer a vector variable directly, and it
bypasses every bound the command declared. A rebuild should not reproduce it; it should
offer a set-with-validation instead, and change those call sites.

The type tests are runtime downward casts in the original. Incidental: a rebuild tags each
command with its kind and switches on the tag.
