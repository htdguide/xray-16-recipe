# src/xrGame/game_cl_base_weapon_usage_statistic.cpp

> Match telemetry: each client tracks its own shots and provisional hits, the server confirms or denies each one, and the confirmed record is periodically pulled back to the server to be written out.

**Needs** — [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`Level.h`](Level.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`Weapon.h`](Weapon.h.md) · [`Actor.h`](Actor.h.md) · [`Hit.h`](Hit.h.md) · [`xrServer.h`](xrServer.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md)
**Tier floor** — T1: packs records to a byte budget and back-patches counts into a packet

## Purpose

The match's telemetry, and it is more interesting than a counter set because it solves a real
distributed problem: **the client knows what it shot at, but only the server knows what it
hit.** A client fires, predicts a hit locally, and records it provisionally; the server runs
the authoritative test and answers yes or no per projectile; only then does the client's
record become final. A separate pull, every thirty seconds, drags the finalized records back
to the server, which writes them out.

That two-phase shape is why almost every record carries a *completion* flag and why every
counter has a *delta* companion.

## State

```text
RECORD TrackedBullet              # a projectile in flight, on the client that fired it
  firer_name   : text
  weapon_name  : text
  bullet       : projectile       # a copy, including its identifier
  hit_refs     : int              # hit reports we sent for it
  hit_responds : int              # answers the server has returned
  removed      : bool             # the projectile itself is gone

RECORD Hit
  from, to     : position         # the segment the projectile travelled on this hit
  bone_id      : int (16-bit, signed)
  bone_name    : text
  target_id    : int (16-bit)
  target_name  : text
  bullet_id    : int
  deadly       : bool
  count        : int (8-bit)      # how many identical hits this row stands for
  completed    : bool             # the server has answered

RECORD WeaponStats
  name, inventory_name : text
  bought               : int
  rounds_fired, bullets_fired, hits, kills : int   # each with a delta companion
  explosion_kills, bleed_kills             : int (16-bit)
  purchase_histogram   : int[3][34]        # [team index][money bucket]
  hits                 : list<Hit>

RECORD PlayerStats
  name, digest : text             # digest identifies the account across matches
  profile_id   : int
  total_shots  : int              # with a delta companion
  alive_time, money_earned, respawns, objectives : per team index
  special_kills: int[4]           # headshot, backstab, knife kill, eye shot
  current_team : int
  last_alive_stamp : int
  money_this_life  : int
  weapons      : list<WeaponStats>

RECORD Collector
  collecting   : bool
  in_flight    : list<TrackedBullet>
  players      : list<PlayerStats>
  totals       : per team index
  last_report  : int
  report_period: int              # 30 seconds
  pending      : list<PendingChecks>   # server side: per client, the hits awaiting a verdict
  lock         : mutex
```

**Invariants**

- **A tracked projectile is released only when it is both removed and fully answered.** Either
  condition alone leaves it in the list. This is the rendezvous between the physics side
  (which knows when a projectile stops existing) and the network side (which knows when the
  server has ruled on it); getting it wrong either leaks the record or finalizes a hit the
  server has not confirmed.
- **Every delta counter is reset by the act of transmitting it**, so a counter is reported
  exactly once. The rounds-fired delta is computed differently from the others — it is a
  difference against the running total rather than an independently incremented counter — and
  is then re-baselined. That asymmetry has no evident reason; the effect is the same.
- Every per-team array is indexed by a *converted* team index, never by the raw team number.
  See the conversion below.
- A player record is found by name, and created on the spot if absent. Nothing ever removes
  one, so a player who leaves keeps his record for the rest of the match — which is the point,
  since the report is written at the end.

## the team index conversion

**Contract** — maps a player's team to an index into every three-wide array. The mapping
differs by mode.

```text
FUNCTION team_index(team) -> int
  t = the mode's own team remapping of `team`
  IF the mode is team deathmatch THEN
    IF t is invalid THEN RETURN 1          # spectators are counted as team 1
    RETURN t                               # 0 and 1 are the two teams; slot 2 unused
  ELSE
    IF t is the spectator team OR invalid THEN RETURN 0
    RETURN t + 1                           # slot 0 is "no team"; the teams shift up
```

**Invariants** — this is why the arrays are three wide and why the third slot means different
things in different modes. In team deathmatch the slots are (team one, team two, unused); in
every other mode they are (no team, team one, team two). Two modes therefore write the same
array with two different conventions, and the report reader must know which mode produced it.
A rebuild should pick one convention.

Attributing spectators to team one in team deathmatch is a fallback with no justification in
the source; it skews that team's alive time by whatever the spectators accumulate.

## `OnBullet_Fire`

**Contract** — called for every projectile leaving a weapon. Records it only if it is a
send-hit projectile fired by an *actor* — so creature and turret fire is not counted.

```text
FUNCTION on_fire(bullet, cartridge)
  RETURN unless collecting, the bullet allows hit reporting,
         its weapon and its parent both resolve, and the parent is an actor
  player = record for the parent's name
  bullet.id = player.total_shots        # the shot ordinal becomes the projectile's identity
  player.total_shots += 1
  player.total_shots_delta += 1
  weapon = player's record for the weapon's section
  weapon.bullets_fired += 1
  weapon.rounds_fired = weapon.bullets_fired / cartridge.pellets_per_round
  weapon.bullets_delta += 1
  in_flight.append(a tracked record for this projectile)
```

**Invariants** — **the projectile's identity is the shooter's shot ordinal.** That is what
makes the identifier unique per client without coordination, and it is why the server's
verdict can name a projectile by a bare number. It also means identifiers are only unique
*per client*, which is fine because the verdict travels back to the client that asked.

Rounds fired is *derived* from projectiles fired divided by the pellet count, so a shotgun
firing one round of eight pellets counts as one round and eight projectiles. Integer
division means a partially delivered round rounds down.

## `OnBullet_Hit`

**Contract** — the client's provisional hit. Only the **first** hit report for a projectile is
recorded; later ones increment the reference count and nothing else.

```text
FUNCTION on_hit(bullet, target_id, bone, location)
  RETURN unless the bullet allows hit reporting and is tracked
  IF this is the first hit report for it THEN
    weapon.hits += 1 ; weapon.hits_delta += 1
    RETURN unless the target resolves AND is an actor
    record a hit: the projectile's position as the start, the hit location as the end,
      the bone and its debug name, the target and its name,
      not deadly, not completed, count 1
    add it through the deduplicating append
  increment the projectile's hit-reference count
```

**Invariants** — counting only the first report is what stops one projectile passing through
several bones from counting as several hits. The reference count still rises, because the
server will answer once per report and the release rule compares the two.

The bone *name* comes from the model's debug name table, which exists only because the
telemetry wants a human-readable report. A rebuild that strips debug names must keep this one
or number the bones.

Non-actor targets record no hit but still count toward the weapon's hit total, so shooting
scenery inflates the hit count without producing a hit row.

## the deduplicating hit append

**Contract** — before appending, scans backwards over the last thirty hits for one with the
same bone, the same target, and both endpoints within half a metre; if found, increments that
row's repeat count instead of appending. Rows cap at 254 repeats.

**Invariants** — this is a compression, and both its bounds are tuning. Thirty is "a
magazine", on the argument that a burst into one target is the case worth collapsing. Half a
metre is the tolerance within which two hits are "the same place". A rebuild is free to
choose differently; what it must keep is that a row's repeat count is part of the format, so
a reader must multiply.

## the server-side check

**Contract** — three calls on the server, in a strict order per hit: a request is queued, a
verdict is attached to the most recent request, and the accumulated verdicts are sent back
per client.

```text
FUNCTION on_check_request(hit)        # server only
  find or create the pending list for hit.sender
  append (hit.bullet_id, hit.bone) to it
  remember this sender as "the last one"

FUNCTION on_check_result(verdict)     # server only
  RETURN unless a last sender is remembered
  attach the verdict to that sender's MOST RECENT request, mark it processed
  count it toward that sender's true or false tally
  forget the last sender
```

**Invariants** — the verdict is attached to *the most recent request of the last requester*,
with no identifier matching. The two calls are therefore a **hidden pair**: the caller must
issue a request and then immediately its result, with no other request in between, from one
thread. Nothing enforces it. This is the most fragile interface in the group, and a rebuild
should pass the verdict with the projectile identifier.

## `Send_Check_Respond`

**Contract** — for each client with processed verdicts, packs them into one message and sends
it. Denials carry only the projectile identifier; confirmations carry the identifier and the
bone the server decided was hit. Each message is prefixed by the two tallies.

**Invariants** — the confirmed bone may differ from the one the client reported, which is the
whole reason a confirmation carries it: the client's prediction is corrected.

Processed entries are removed from the pending list by swapping the last one into their place,
so the list's order is not preserved. It does not need to be: each entry names its own
projectile.

**Notes** — the two payload runs are assembled in raw stack buffers through pointer walks and
appended wholesale, with no bound on how many verdicts accumulate between sends. A burst
larger than the buffers overruns them. A rebuild writes the entries into the packet directly
and splits when the packet fills.

The tallies written into the message are the *running* counters, which are reset on send,
while the entries written are only the processed ones — and the source carries a commented-out
line comparing the two, suggesting they were observed to disagree. A rebuild should write the
count of entries actually packed.

## `On_Check_Respond`

**Contract** — the client applies the verdicts. Each answer increments the projectile's
respond count and then tries to release it. A confirmation additionally scores a kill for the
weapon, marks the matching hit row deadly, and corrects its bone to the server's.

**Invariants** — a confirmed hit is a **kill**, not merely a hit: this exchange exists to
settle lethality, not impact. The hit count was already credited optimistically at hit time.

A verdict for an unknown projectile is warned about and skipped — it happens when the
projectile was released before its answer arrived.

## `RemoveBullet`

**Contract** — the release rendezvous. Does nothing unless the projectile is both removed and
fully answered. Otherwise marks its hit row complete and drops the tracked record by swapping
the last one into its place.

## the event hooks

Each is a one-line attribution, all guarded by the collecting flag and a non-null player
record:

- **weapon bought** — increments the purchase count and one bucket of the histogram. The
  bucket is zero below five hundred money and otherwise one per thousand; both the team index
  and the bucket are bounds-checked and the sample is dropped rather than clamped when either
  is out of range.
- **player spawned** — counts a respawn, zeroes the money-this-life accumulator, records the
  team, and stamps the alive-time clock.
- **money added** — accumulates into money-this-life; negative amounts are ignored.
- **player killed** — moves money-this-life into the per-team money total, both per player and
  for the match.
- **objective delivered** — increments the per-team objective count.
- **special kill** — increments one of four counters, selected from the special kill type.
  Only four of the eight special kinds are counted; the streak, promotion and scanner kinds
  have no counter.
- **explosion kill** and **bleed kill** — server-side attributions that bypass the projectile
  tracking entirely, because neither has a projectile. Each credits a hit and a kill to the
  killer's weapon and appends an already-complete, already-deadly hit row. The explosion row
  stores the killer's and the weapon's positions in the segment fields, which is not a
  trajectory; the bleed row stores zeroes. Both are placeholders a report reader must know to
  ignore.

## `SVUpdateAliveTimes`

**Contract** — server only. Walks every connected client and, for each whose player is not
permanently dead and has a name, adds the elapsed wall time since its last stamp to that
player's per-team alive time and re-stamps. Then recomputes the three match totals by summing
every player.

**Invariants** — alive time accumulates against *wall* time, not server time, and only for
players the server still holds a client record for. A player who disconnects stops
accumulating, correctly, but his last partial interval is lost.

The totals are recomputed from scratch each tick rather than maintained incrementally, which
is O(players) per tick and correct by construction.

## `Update`

**Contract** — per scheduled tick. Updates alive times always; on the server, every thirty
seconds broadcasts a request for every client to report.

## `OnUpdateRequest` / `OnUpdateRespond`

**Contract** — the pull. A client answers with its own player record; the server merges the
answer into its copy of that player, also recording the account digest and profile identifier
the transport attached to the sender.

**Invariants** — a client reports **only its own** record, which is what makes the protocol
tamper-limited: a client can lie about its own numbers and about nothing else. The digest and
profile identifier come from the transport, not the client, so the report is bound to an
account the client cannot choose.

The merge is *additive*: received deltas are added to the server's running totals. That is
what makes repeated pulls compose, and it is why the client resets its deltas on send.

## the two compression dictionaries

**Contract** — a transmitted player record is preceded by two tables built from its own hit
rows: victim names, indexed by a byte, and bone (name, identifier) pairs. Hit rows then carry
a one-byte victim index and the bone identifier, instead of two strings each.

**Invariants** — the tables are per *transmission*, built fresh from whatever hits are being
sent. The victim table caps at 255 entries and refuses further names; the bone table refuses
a name or an identifier it already holds, so one bone is never listed twice under either key.

A victim name the table does not contain resolves to **index zero**, which names the first
victim — a silent misattribution rather than an error. It cannot happen while the table is
built from the same rows being sent; it is a latent hazard if the two ever diverge.

An out-of-range victim index on load returns an empty name, but the bound is tested with the
wrong comparison — an index exactly equal to the size passes — so a malformed packet reads
one past the end.

The two tables' scratch storage is allocated on the stack for a fixed 255 victims and 65
bones. Sixty-five is one more than the largest skeleton in the shipped data; the tables are
built from hits and so cannot exceed the skeleton's bone count.

## the packet budget

**Contract** — two places stop writing when the packet is nearly full: a player record refuses
to write *anything* if its two dictionaries plus one weapon record would not fit, and a
weapon record stops emitting hit rows when the next one would not fit.

**Invariants** — the hit filter is a **destructive** read: every hit it writes is removed from
the list as it goes, so a hit is sent once and then forgotten. Hits it could not fit stay,
and go in the next report. That is the mechanism by which the report streams over many pulls.

The all-or-nothing behaviour at the player level means a player whose dictionaries alone
overflow a packet **never reports**, silently. With 255 victims and a long bone table that is
reachable. A rebuild should split the report across packets.

**Notes** — the hit count is written as a placeholder and back-patched after the rows are
emitted, because the number that fit is not known until they are written. That is the right
shape; a rebuild needs the same ability to seek back in a partially written packet.

Two fields — explosion kills and bleed kills — are serialized out on both sides, with the
comment that the server sets them itself. They are therefore server-local and do not survive
a client's report.

## `SetCollectData`

**Contract** — turns collection on or off; turning it *on* clears everything first, so a match
never inherits the previous one's numbers. Turning it off preserves what was collected, which
is what lets the report be written after the match ends.

## the lock

**Contract** — a scoped guard taken on entry to most mutating calls.

**Invariants** — the coverage is not uniform: the projectile-removal hook, the server-side
check trio, the response send and several attribution hooks take no lock while touching the
same lists. Given that hits arrive on the network thread and updates run on the simulation
thread, those are real races. A rebuild should either lock uniformly or confine the collector
to one thread and queue events into it.

Several locked calls also call other locked calls — the player lookup takes the lock and is
reached from callers that already hold it — so the lock must be recursive. A rebuild with a
non-recursive mutex must split each operation into a locked outer and an unlocked inner.
