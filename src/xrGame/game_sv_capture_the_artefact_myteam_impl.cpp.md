# src/xrGame/game_sv_capture_the_artefact_myteam_impl.cpp

> The per-team objective record: one team's artefact, its home point, who is carrying it, since when it has been loose, and whether somebody has activated it — plus the three searches the mode runs over the pair of them.

**Needs** — [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`Level.h`](Level.h.md)
**Used by** — [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md)
**Tier floor** — T3: a record and its transitions; no algorithm beyond a minimum search

## Purpose

Split out of [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) for
compilation reasons, and a rebuild folds it back. It holds one record and six transitions on
it, and the record is worth its own page because **it is the mode's entire model of the
objective**: everything the delivery, returning and ownership rules ask is a question about
one of these fields.

Exactly two instances exist, one per playing team, held in a map keyed by team.

## State

```text
RECORD TeamState
  team_list_index    : int                # where this team's money and skin record sits
  players_count      : int (16-bit)       # used ONLY by the "any team" assignment
  score              : int (signed)       # deliveries made BY this team
  team_name          : text               # the configuration section it was loaded from
  artefact_name      : text               # the class this team's artefact is created from

  artefact           : entity             # this team's artefact; created once per round
  artefact_owner     : entity             # who is CARRYING it; none when it is not carried
  home_point         : (position, angles) # authored into the level
  home_point_set     : bool               # the load-time completeness check

  loose_since        : int (ms)           # server clock; 0 means "not loose"
  activated          : bool
  activated_since    : int (ms)           # server clock; 0 when not activated
  last_activator     : int (16-bit)       # entity identifier; 0 when not activated
```

Invariants:

- **`artefact_owner` being absent is the single test for "this artefact is loose".** It is set
  on attach and cleared on detach, and nothing else writes it. Both the returning check and
  the delivery check begin by asking it.
- **The score on this record counts deliveries, not frags**, and it is the number the round-end
  check compares against the score limit. Kills never touch it.
- **`loose_since` and the activation triple are mutually exclusive in practice**: attaching
  clears both, detaching sets the first, activating sets the second. A rebuild can model the
  artefact's situation as one tagged value — *at home*, *carried by X*, *loose since T*,
  *activated by X at T* — and the flags collapse.
- **`home_point_set` exists only so that loading can fail loudly.** A team whose level never
  authored an artefact point has no home to deliver to and the mode cannot run; the flag is
  checked once, after the level's point chunk is read, and never again.
- **`players_count` is incremented and never decremented.** It is touched only when a player
  asks for "any team", so the automatic assignment drifts as players join and leave. See
  [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md); the between-rounds
  balancing pass counts clients properly and repairs it, but within a round it does not. A
  rebuild should count the clients and delete this field.

**Notes** — the record is copied by value into the team map at load, and the copy is written out
field by field by hand. It reproduces every field, including the two raw entity references,
which the copies then share. That is an artifact of the language rather than a decision: the
problem it solves is only "this aggregate must be duplicable while holding references it does
not own", and a rebuild whose records are plain values with handles instead of pointers has
nothing to write.

## `SetArtefactRPoint`

**Contract** — records this team's authored home point and marks the team as having one.

**Invariants** — the flag is set here and only here, so the load-time completeness check
below is exactly "did the level name a point for this team".

## `OnPlayerAttachArtefact`

**Contract** — somebody picked the artefact up. Records the carrier and clears everything about
its previous idle state: the loose timestamp, the activation timestamp and the activation flag.

```text
FUNCTION on_player_attach_artefact(new_owner)
  artefact_owner  = new_owner
  loose_since     = 0
  activated_since = 0
  activated       = false
```

**Invariants** — **picking an artefact up cancels a pending return.** An activated artefact that
somebody picks up before the returning check runs is carried, not teleported. That is the right
resolution of the race: a player physically holding the objective beats a timer.

Clearing the loose timestamp is what makes the returning clock measure *uninterrupted* idleness
rather than time since the artefact left home. An artefact repeatedly picked up and dropped in
a contested corridor never returns itself.

## `OnPlayerDetachArtefact`

**Contract** — the carrier lost it. Clears the owner and stamps the moment it became loose,
which starts the returning clock.

**Invariants** — the routine asserts that the player detaching is the player recorded as the
owner. That is the ownership invariant stated as a check: at most one player carries an
artefact at any time, and the only way to stop carrying it is to be the one carrying it.

**Notes** — the activation state is deliberately **not** cleared here. An artefact that was
activated, then picked up, then dropped, has already had its activation cleared by the attach.
An artefact that was never picked up cannot reach this routine. So the omission is safe by
construction rather than by check — worth stating, because a rebuild that reaches this
transition by a different path must clear it.

## `OnPlayerActivateArtefact` / `IsArtefactActivated` / `DeactivateArtefact`

**Contract** — a player triggered his own team's displaced artefact. Records the moment, raises
the flag, and remembers who did it so he can be paid when the artefact reaches home. Asking
reads the flag; deactivating clears all three.

```text
FUNCTION on_player_activate_artefact(who)
  activated_since = server_clock
  activated       = true
  last_activator  = who

FUNCTION deactivate_artefact()
  activated       = false
  activated_since = 0
  last_activator  = 0
```

**Invariants** — **only one activator is remembered**, the most recent. Two defenders reaching
the artefact together means only the second is paid. The record is a single slot because the
reward is a single payment, not a share.

**Notes** — `activated_since` is written and read nowhere. It was the input to a delay between
activating an artefact and its arriving home, and that delay is commented out at the call site
in [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md): activation is
instantaneous as shipped. Whether an escort period was intended is not recoverable, but the
field's existence is the evidence that it was designed.

The activator is an **entity** identifier, so the payment path has to resolve it back to a
player and tolerate his having died in between. That tolerance is real: a defender who
activates the artefact and is shot a moment later simply is not paid.

## `GetArtefactOwner`

**Contract** — the carrier, or nothing.

## The three searches

**Contract** — three predicates used to search the pair of team records. Each answers one
question the ownership rules ask constantly.

```text
FEWEST_PLAYERS(a, b)     -> a has fewer players than b       # the "any team" assignment
ARTEFACT_IS(record, id)  -> record's artefact exists and has that identifier
OWNER_IS(record, id)     -> record's owner exists and is that actor
```

**Invariants** — both identity searches **test for existence before comparing**, so a team whose
artefact has not spawned yet — the window between round start and the artefact respawn — never
matches. Without that guard an unspawned artefact would compare equal to whatever an absent
reference reads as, and every touch in the level would be treated as touching the objective.

**Notes** — the ownership search is how the mode answers "is this player carrying an artefact,
and whose?" It is run on every touch, on every death and on every disconnect, over two records.
With two teams a linear scan is the right implementation and a rebuild should not index it; the
reason to name the search at all is that it is the *question*, not the loop.

The fewest-players comparison is written as a strict less-than, so a tie resolves to whichever
team the map iterates first — green. A tie is the common case on an even server, and the effect
is that "any team" sends players to green until it is ahead. Combined with the counter never
being decremented (see State), the automatic assignment is best understood as a hint the
balancing pass corrects, not as a rule.
