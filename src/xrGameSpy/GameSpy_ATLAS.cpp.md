# src/xrGameSpy/GameSpy_ATLAS.cpp

> Statistics and persistent player storage: authorise a session against a logged-in
> profile, build a structured match report, and submit it. This is what the game records
> about a match and about a player.

**Needs** — [`GameSpy_ATLAS.h`](GameSpy_ATLAS.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [`GameSpy_GP.h`](GameSpy_GP.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_ATLAS.h`](GameSpy_ATLAS.h.md)
**Tier floor** — T2: an asynchronous channel with a builder. Nothing needs manual layout.

## Purpose

Two jobs that share a service and are otherwise unrelated:

1. **Authorise** a logged-in profile for statistics, yielding the credential pair that
   every later submission carries.
2. **Report** a finished match, as a structured document with a global section, a
   per-player section, and typed key/value entries within each.

It sits behind the account service, not beside it: a profile must be logged in before it
can be authorised here, and the game's login is not considered complete until it is (see
[`GameSpy_GP.cpp`](GameSpy_GP.cpp.md)).

## State

```text
RECORD StatisticsChannel
  interface : ChannelHandle       # opened against the title's numeric id; one per process

# What authorisation yields, held by the game on the logged-in profile.
RECORD ReportingCredential
  certificate  : opaque           # presentable, identifies the profile to the service
  private_data : opaque           # the secret half; signs submissions

# Per report
RECORD MatchReport
  header_version : int
  players        : int
  teams          : int
  connection_id  : bytes (fixed-width)   # identifies this *server's* reporting session
  sections       : global, then per-player
  entries        : list<(field_id : int (16-bit), value : int (32-bit) or text)>
  status         : {complete, incomplete, ...}
  authoritative  : bool                  # see the invariant
```

**Invariants**

- A report's player count and team count are fixed when the report is created. Every
  per-player section must be filled; the document is positional, not keyed.
- Every submission carries the `authoritative` flag, which says whether the submitter is
  the match's authority (the server) or a participant (a client). The service uses it to
  decide whose numbers to believe when both report. A rebuild that only ever lets the
  server report can drop the flag — but must then make sure clients cannot report.
- The connection id identifies the *reporting session*, not a player; each player's
  section carries its own connection id and profile id separately.

## `Init` · construction

**Contract** — opens a channel against this title's numeric id. Construction calls it and
logs a failure; there is no retry and no flag recording that the channel is dead, so every
later call proceeds against a null handle.

## `Think`

**Contract** — one poll per frame. Delivers completed submissions and authorisations.

## `WSLoginProfile`

**Contract** — authenticates `(nick, email, password)` against the statistics service —
the *same* credentials already used to log in to the account service — and yields the
certificate and private-data pair. Non-blocking, completing through a callback that
carries both a transport result and a login result, each of which can fail independently.

**Notes**

The second authentication is the part a rebuild must decide about deliberately. Its
purpose is separation: the account service holds who you are, the statistics service holds
what you did, and the certificate is what lets the second trust the first without the two
sharing a password store. A rebuild with one service behind both does not need it. A
rebuild that keeps them separate needs exactly this: **a credential, obtained once per
login, that authorises a client to submit statistics as a particular profile.**

The game sends the password to *both* services. A rebuild should get the certificate from
the account service's ticket instead of re-presenting the password.

## `CreateSession` · `SetReportIntention` · `GetConnectionId`

**Contract** — `CreateSession` opens a reporting session for a credential; the session's
identifier is read back with `GetConnectionId`. `SetReportIntention` declares, before a
match, that this participant intends to report and whether it will do so authoritatively.
All non-blocking, each with a millisecond timeout and a completion callback.

**Notes** — declaring intent before the match is not ceremony: it is what lets the service
know a report is coming and reconcile the several reports of one match. Stated as a
capability: *before a match, every participant that will report announces itself; after
it, each submits; the service reconciles.* A rebuild with a single authoritative reporter
does not need the announcement.

## Building a report

**Contract** — a sequence of builder calls, in a fixed order, each returning a result
code. Nothing here validates the order; calling them out of order produces a malformed
document.

```text
FUNCTION build_report(players : int, teams : int) -> MatchReport
  report <- create_report(header_version, players, teams)
  begin_global_section(report)
    add_int(report, field_id, value)          # repeated
    add_text(report, field_id, value)         # repeated
  begin_player_section(report)
  FOR EACH player
    begin_new_player(report)
    set_player_data(report, player_index, player_connection_id, team_index,
                    outcome, profile_id, certificate)
    add_int / add_text ...                    # this player's statistics
  end_report(report, authoritative, status)
```

**Invariants** — `set_player_data` binds a section to a *profile*, so an unauthenticated
player's section has no profile id and its numbers attach to nobody. The per-player
authentication data that would normally accompany it is passed as **sixteen zero bytes** —
the code explicitly zeroes a buffer and submits that. Per-player authentication is
therefore not performed: the submitter's credential covers the whole report. A rebuild
should either do the same knowingly or authenticate each player properly, and the zeroed
buffer is the marker that the original chose neither.

**Could not recover** — the field ids used inside a report. Nothing in this repository
writes one: the report builder is fully wrapped here and called from nowhere. The
statistics *vocabulary* that survives is on the reading side instead — see below.

## `SubmitReport`

**Contract** — submits a finished report under a credential, with a timeout and a
completion callback. Non-blocking.

## `TryToTranslate`

**Contract** — three overloads, for a transport failure, an authorisation failure and a
channel failure. Each **discards its argument** and returns one fixed localization key per
category. The player learns which of three things broke and nothing more.

**Notes** — the argument is taken and ignored in all three. That is a deliberate
simplification, not an oversight: the service's own failure codes were not worth
translating into the string table. A rebuild should keep the three categories and log the
detail.

## What is actually recorded, and when

The report builder above is dead code in this repository — no call site constructs a
report. What survives is the *shape of the data the game expects a statistics service to
hold about a player*, which is declared on the reading side and is the useful half:

```text
RECORD PlayerPersistentRecord
  awards      : map<AwardKind, (times_earned : int, last_earned : date)>
  best_scores : map<ScoreKind, int>

ENUM AwardKind          # 30 of them, earned in a match and accumulated across matches:
  massacre, paranoia, overwhelming_superiority, blitzkrieg, dry_victory,
  multichampion, mad, achilles_heel, faster_than_bullets, harvest_time, skewer,
  double_shot_double_kill, climber, opener, toughy, invincible_fury, oculist,
  lightning_reflexes, sprinter_stopper, marksman, peace_ambassador, deadly_accuracy,
  remembrance, avenger, cherub, dignity, stalker_flair, lucky, black_list, silent_death

ENUM ScoreKind          # personal bests, each a run length within one match:
  kills_in_row, knife_kills_in_row, backstabs_in_row, head_shots_in_row,
  eye_kills_in_row, bleed_kills_in_row, explosive_kills_in_row
```

**When**: a report is created and submitted at the end of a match, by the participants
that declared intent, and the service accumulates awards and maximises best scores across
matches. Both record sets are read back at login and exposed to the game's scripts, which
is what makes them load-bearing — **the shipped Lua scripts query a player's awards and
best scores**, so a rebuild must keep the two record shapes and the enumerations, even if
nothing ever fills them.

**The null path.** The engine's profile store — the object the game and the scripts read
these records through — is already a **stub in this repository**: it holds an empty award
map and an empty best-score map, and its "load my profile" operation does not reach the
network at all. It succeeds if a profile is logged in and otherwise fails with the
localization key `mp_first_need_to_login`. So the shipped behaviour with no statistics
service is: *every player has zero of every award and a zero personal best, the scripts
that read them see empty maps and work, and no match is ever reported.* This is the
clearest example in the chapter of a null implementation that is already written — a
rebuild should copy it rather than design one.
