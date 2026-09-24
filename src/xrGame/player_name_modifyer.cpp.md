# src/xrGame/player_name_modifyer.cpp

> Replaces every character in a player-chosen nickname that would break a file path or a console line.

**Needs** — [`player_name_modifyer.h`](player_name_modifyer.h.md)
**Used by** — [`player_name_modifyer.h`](player_name_modifyer.h.md)
**Tier floor** — T2: a byte-wise scan over a fixed buffer

## Purpose

A nickname is player-supplied text that ends up in three unsafe places: a save-file path, a
console command line, and a format string. Four characters break one of those, and each is
replaced with an underscore.

The function is worth a page only because of what the forbidden set says.

## State

`Stateless.`

## `modify_player_name`

**Contract** — copies the given name into the caller's buffer, replacing every forbidden
character with an underscore, and reports the buffer. The length is the same; nothing is
removed and nothing is inserted. The buffer is a fixed 256 bytes and the input is **not
length-checked** — the caller must have validated it.

```text
FUNCTION modify_player_name(source, out destination) -> text
  destination = source
  FOR EACH position IN destination
    IF destination[position] IN forbidden
      destination[position] = "_"
  RETURN destination
```

**Notes** — the forbidden set is the path separator, a question mark, a percent sign and a
double quote. Each has a distinct reason:

- the **path separator**, because the nickname becomes part of a save-file path and a
  separator would escape the intended directory;
- the **question mark**, because the platform's file system rejects it in a name;
- the **percent sign**, because the nickname is passed through format strings for the
  console and the score display, and an unescaped format directive would read arbitrary
  memory;
- the **double quote**, because a console line is tokenized on quotes and a nickname
  containing one would split into two arguments.

Only the path separator is taken from the platform; the other three are literal. A rebuild
on a system with a different separator must substitute it, and must forbid both separators
if it ever writes paths for the other system.

The scan advances its search origin by one position after each replacement rather than
continuing from the replaced position. Because the replacement is itself never a forbidden
character, the effect is the same as a plain scan; the extra bookkeeping costs nothing and
means nothing.

The percent sign is the security-relevant one. A rebuild whose formatting is
type-safe does not need it forbidden — but the nickname is also written back out to
configuration and read by the original engine, so removing it from the set changes what the
original will accept.
