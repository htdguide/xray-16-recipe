# src/xrGame/cdkey_ban_list.cpp

> The multiplayer server's persistent list of banned players, keyed by the hash of a player's product key.

**Needs** — [`cdkey_ban_list.h`](cdkey_ban_list.h.md) · [`xrServer.h`](xrServer.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a list, a text file and a wall-clock comparison

## Purpose

A server operator needs a ban to survive a reconnect, a name change and a restart. Banning
by network address fails the first two; banning by display name fails all three. So a ban is
keyed by the hash of the player's product key — the one identifier a client presents that it
cannot trivially change — and the list is kept as a text file beside the server's other
settings, readable and editable by hand.

The file also records who issued each ban and from where, because a server with several
administrators needs to know which one to ask about a contested ban.

## State

```text
RECORD BannedClient
  client_digest : text        # hex hash of the product key -- the actual ban key
  client_ip     : text        # informational only; never matched against
  client_name   : text        # display name at the moment of the ban; informational
  ban_start     : timestamp   # wall clock, local time
  ban_end       : timestamp   # wall clock, local time -- invariant: > ban_start
  admin_digest  : text        # who banned them; empty means the server itself
  admin_ip      : text
  admin_name    : text        # "Server" when nobody in particular

RECORD cdkey_ban_list
  entries : list<BannedClient>
```

**Invariants**

- The digest is the only field a ban is matched on. Address and name are recorded so a human
  can read the list; changing either does not lift a ban, and matching on either would ban
  the wrong person.
- An entry with no digest or no end time is not a ban and is rejected at load. Every other
  field is optional and defaults to blank.
- The in-memory list and the file are kept in step by rewriting the whole file after every
  mutation. The list is tens of entries at most, so there is no reason for anything cleverer,
  and a server killed between a ban and a flush would otherwise forget it.
- Expiry is *lazy*: expired entries are removed when the list is loaded and again on every
  lookup, not by a timer. A ban that expired while the server was down is therefore gone the
  first time anyone connects.

## the ban file

**Contract** — a configuration file named `banned_list.ltx` under the application's data
root. Each ban is one section; section names are positional (`client_0`, `client_1`, …) and
carry no meaning, being regenerated from scratch on every write. Eight keys per section:
client digest, ban start, ban end, client name, client address, administrator name,
administrator address, administrator digest.

Timestamps are written as local wall-clock text, `DD.MM.YYYY_HH:MM:SS`.

**Notes** — the timestamp format is the single most fragile thing here. It carries no time
zone and no daylight-saving marker, and it is parsed back with the same fields, so a server
whose machine changes zone shifts every unexpired ban by the difference. That is a real
defect a rebuild should fix by storing an absolute instant; the human-readable form belongs
in the printout, not the file.

**Notes** — the format tolerates a missing start time, name, address or administrator field,
so an operator can write a ban by hand with only a digest and an end time.

## `load`

**Contract** — reads every section of the ban file into the list, rejecting and logging any
section that lacks the two required keys, then drops everything already expired. A missing
file yields an empty list rather than an error: a server that has banned nobody has no file.

## `save`

**Contract** — rewrites the whole file from the list, numbering sections by position.
Called after every mutation and at destruction.

**Notes** — the file is opened for write with the reader's "do not load the existing
contents" behaviour, so the rewrite is a truncate-and-write, not a merge. Two server
processes sharing one ban file will clobber each other; the design assumes one.

## `is_player_banned`

**Contract** — given a connecting client's digest, first drops expired entries, then scans
for a matching digest. Reports the banning administrator's name alongside the verdict, so
the refusal message can tell the player who to argue with. A missing digest is not a ban —
a client that presents no product key is refused elsewhere, not here.

```text
FUNCTION is_player_banned(digest) -> (bool, admin_name)
  IF digest is empty THEN RETURN (false, "")
  erase_expired()
  FOR EACH e IN entries
    IF e.client_digest == digest THEN RETURN (true, e.admin_name or "Server")
  RETURN (false, "")
```

## `ban_player`

**Contract** — bans a connected client for a duration in seconds, attributing it to an
administrator or to the server itself. Refuses two cases outright:

- **A client with administrator rights cannot be banned.** This is the protection against
  one administrator locking out another, and against a compromised command banning the
  operator off their own server.
- **A client with no digest cannot be banned**, because there would be nothing to match
  later. The refusal explicitly directs the operator to an address ban instead, which is a
  different mechanism.

The entry records the client's current address and display name for the record, stamps the
start as now and the end as now plus the duration, and persists immediately.

**Notes** — the administrator's own digest is recorded in the entry. The source's comment
about "bad admins" is the reason: a ban is attributable, so an operator reviewing the file
can tell which administrator issued it even if the display name was changed.

## `ban_player_ll`

**Contract** — the same, but from a digest alone rather than a connected client. This is how
an operator bans someone who is not currently connected, and how a ban is applied from a
console command or a script. Address and name are recorded as unknown, since there is
nobody to ask. The administrator-rights check is absent here — the caller is already
privileged.

## `unban_player_by_index`

**Contract** — removes the entry at a position in the list and persists. The index is the
one printed by the listing, which is why the listing numbers its rows. An out-of-range index
is reported and ignored.

**Notes** — indices are positional and shift when anything is removed or expires. An
operator who lists, then waits, then unbans by the index they read may remove the wrong
entry. A rebuild should key the removal on the digest.

## `print_ban_list`

**Contract** — prints one line per entry — index, address, name, expiry and digest — with an
optional substring filter applied to the whole formatted line. Filtering the *rendered* line
rather than a chosen field is deliberate: it lets an operator search by any of them with one
command.

## `erase_expired_ban_items`

**Contract** — removes and logs every entry whose end time is in the past, comparing against
the current wall clock. Called at load and at every lookup, which is enough: the only thing
that observes the list is a lookup or an operator's listing, and the listing's staleness
does not matter.
