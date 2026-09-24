# src/xrGameSpy/GameSpy_Keys.h

> The catalogue of every field a server advertises about itself, its players and its
> teams. This is the request/response shape of the server list, and it is the most
> load-bearing page in the chapter.

**Needs** — [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_QR2_callbacks.cpp`](../xrGame/gamespy/GameSpy_QR2_callbacks.cpp.md) · [`xrGameSpyServer_callbacks.h`](../xrGame/xrGameSpyServer_callbacks.h.md) · [`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md) · [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md)
**Tier floor** — T3: it is a numbering. The numbers are a wire identity, so they must be
stable, but nothing about holding them constrains the language.

## Purpose

A server advertisement is a flat bag of named values. Both ends need to agree on the
names, and the query protocol carries a compact numeric id instead of the name whenever it
can, so both ends also need to agree on the numbering. This file is that agreement.

The numbering splits in two at 100, and the split is the important thing:

- **1 – 29 are the standard fields** every game on the service advertised. Their meanings
  and names were fixed by the service, not by this game. They cover the fields a generic
  server browser can display without knowing what game it is looking at.
- **100 and up are this game's own fields**, registered at startup by
  [`GameSpy_QR2.cpp`](GameSpy_QR2.cpp.md), which is where each id is bound to the text
  name that actually travels. They describe multiplayer rule settings a player filters
  and chooses on: friendly fire, artefact counts, respawn timing.

Ids 30–99 are unused. The gap is not accidental — the service reserved the first 50 ids
and the file records a cap of 254 registered keys with 50 reserved, so a game's own fields
had to start above the reserve. A rebuild inventing its own protocol has no such
constraint, but it does still need a reserved range if it wants to add standard fields
later.

## State

```text
RECORD FieldCatalogue
  max_registered_fields : int = 254   # the id space a server may register into
  reserved_field_count  : int = 50    # ids below this belong to the service, not the game
```

### Standard fields — the service's own, ids 1–29

These arrive with a server record whether the game asks for them or not. Only the ones
the engine actually reads are marked; the rest are declared for completeness and never
touched.

| id | wire name | meaning | read by the client? |
|---|---|---|---|
| 1 | `hostname` | the server's display name | yes |
| 2 | `gamename` | which title this server belongs to | no — implied by the list queried |
| 3 | `gamever` | the server's build version | yes, shown and compared |
| 4 | `hostport` | the port a client connects on (not the query port) | yes |
| 5 | `mapname` | the level currently running | yes |
| 6 | `gametype` | the game mode's display name | yes, and used as a fallback |
| 7 | `gamevariant` | — | no |
| 8 | `numplayers` | players currently connected | yes |
| 9 | `numteams` | teams in play | yes |
| 10 | `maxplayers` | capacity | yes, defaults to 32 when absent |
| 11 | `gamemode` | lobby state; this engine always answers `openplaying` | no |
| 12–18 | team play, frag and time limits, elapsed/round timing | superseded by this game's own fields | no |
| 19 | `password` | the server requires a password | yes |
| 20 | `groupid` | — | no |
| 21–27 | per-player: name, score, skill, ping, team, deaths, profile id | the per-player record | partly — see below |
| 28–29 | per-team: team name, team score | the per-team record | partly — see below |

### This game's fields — ids 100 and up

Registered by the server at startup; answered per query. Grouped by the layer of the game
that owns the value, which is how the source groups them and is genuinely informative — a
field's group tells you which game modes will have a meaningful answer for it.

| id | wire name | type | meaning |
|---|---|---|---|
| 100 | `gametypename` | int | the game mode as a *number*, not a display string |
| 101 | `dedicated` | bool | the server runs headless |
| 102 | `maprotation` | bool | the server cycles maps at round end |
| 103 | `voting` | bool | players may call votes |
| 104 | `spectatormodes` | int | bit set of the spectator cameras allowed |
| 105 | `fraglimit` | int | kills that end the round |
| 106 | `timelimit` | real | minutes that end the round |
| 107 | `damageblocktime` | real | invulnerability window after respawn |
| 108 | `damageblockindicator` | bool | that window is shown to other players |
| 109 | `anomalies` | bool | anomalies are active in the arena |
| 110 | `anomaliestime` | real | how long anomalies persist |
| 111 | `warmuptime` | real | pre-round warm-up |
| 112 | `forcerespawn` | real | seconds before a dead player is respawned regardless |
| 113 | `autoteambalance` | bool | the server moves players to even the teams |
| 114 | `autoteamswap` | bool | teams swap sides between rounds |
| 115 | `friendlyindicators` | bool | team-mates are marked |
| 116 | `friendlynames` | bool | team-mates' names are shown |
| 117 | `friendlyfire` | real | damage multiplier applied to team-mates |
| 118 | `artefactscount` | int | artefacts in play |
| 119 | `artefactstaytime` | real | how long a dropped artefact remains |
| 120 | `artefactrespawntime` | real | delay before a taken artefact reappears |
| 121 | `reinforcement` | text | see the note below — this one is not a plain number |
| 122 | `shieldedbases` | bool | bases repel the opposing team |
| 123 | `returnplayers` | bool | players are returned to base on round end |
| 124 | `bearercant_sprint` | bool | the artefact carrier may not sprint |
| 130 | `spectator_` | bool | per-player: this player is spectating |
| 131 | `artefacts_` | int | per-player: artefacts carried |
| 132 | `t_score_t` | int | per-team: score |
| 133 | `max_ping_limit` | int | ping above which the server refuses a client |
| 134 | — | — | an anti-cheat presence flag; the server answers it with nothing |
| 135 | `user_password` | bool | the server admits only a listed set of accounts |
| 136 | `server_up_time` | text | how long the server has been running, preformatted |

**Invariants**

- A field id's binding to its text name is fixed at server startup and is global to the
  process. The client resolves an id to a name through the same table when it reads a
  value back, so the two ends agree by construction — but only because they are the same
  binary. **A rebuild must publish this table as protocol, not derive it from shared
  code.**
- Per-player and per-team field names end in an underscore. That is not decoration: the
  query protocol appends the row index to the name, so `spectator_` becomes `spectator_3`
  for the fourth player. Any rebuild that keeps a flat-bag advertisement needs the same
  convention or an explicitly indexed structure.
- Ids 125–129 are burned. The source has them commented out: per-player name, frags,
  deaths, rank and team were going to be this game's own fields before it was realised the
  standard per-player fields (21–27) already carry them. **Do not reuse those ids** — a
  server of this vintage may still have them registered.

**Notes**

*Reinforcement* (121) is the one field whose type is genuinely ambiguous, and the client
reads it twice to resolve that: once as text to see whether it is one of the two sentinel
values `-1` (no reinforcement) or `0` (immediate), and, only if it is neither, again as a
number. A rebuild should make it an explicit tagged value; the double read exists because
the flat bag has no way to say "an enum or a number".

*Game mode* is advertised twice, as a number (100) and as a display string (6), and the
client prefers the number and falls back to parsing the string when the number is absent
or zero. The fallback is what lets this client list servers running the two older titles,
which never registered the numeric field. That compatibility is the reason both exist.

**Could not recover** — what field 134 was for beyond its name suggesting a third-party
anti-cheat; the server registers no name for it and answers it with an empty value, so it
is inert on both ends.
