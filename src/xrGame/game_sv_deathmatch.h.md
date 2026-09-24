# src/xrGame/game_sv_deathmatch.h

> Declares free-for-all deathmatch, which is also the base every other competitive mode narrows: the round machine, the economy, spawn placement and the anomaly rotation, each declared so a descendant can replace it.

**Needs** — [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_deathmatch_process_event.cpp`](game_sv_deathmatch_process_event.cpp.md) · [`game_sv_mp.h`](game_sv_mp.h.md) · [`Hit.h`](Hit.h.md) · [`xrServerEntities/inventory_space.h`](../xrServerEntities/inventory_space.h.md) · [`xrEngine/pure_relcase.h`](../xrEngine/pure_relcase.h.md) · [`xrCore/client_id.h`](../xrCore/client_id.h.md)
**Used by** — [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_deathmatch_process_event.cpp`](game_sv_deathmatch_process_event.cpp.md) · [`game_sv_deathmatch_script.cpp`](game_sv_deathmatch_script.cpp.md) · [`game_sv_teamdeathmatch.cpp`](game_sv_teamdeathmatch.cpp.md) · [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md)
**Tier floor** — T2: a class declaration over session rules; the fixed-width arrays are a layout convenience, not a requirement

## Purpose

The rules are in [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md). What the declaration
alone fixes is the shape every competitive mode inherits, and three things that live nowhere
else: **the respawn-point scoring record and its ordering**, **the fixed team width of the
spawn bookkeeping**, and **the accessor indirection that turns every match limit into an
overridable policy question rather than a variable read**.

This file is the reason the other modes are short. Team deathmatch, artefact hunt and — by
imitation rather than inheritance — capture the artefact all read as differences against what
is declared here.

## State

The operational record and the session settings are in
[`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md). What the declaration itself fixes:

```text
RECORD DeathmatchDeclaration
  free_rpoints   : list<int> [4]      # per team slot — FIXED WIDTH, see Invariants
  last_rpoint    : int       [4]      # per team slot
  teams          : list<TeamScore>    # GROWABLE, and empty in plain deathmatch

  anomaly_names      : list<list<text>>   # authored zone names, per set
  anomaly_ids        : list<list<int>>    # the entities those names resolved to
  anomaly_permanent  : list<text>
  unused_set_ids     : list<int>          # the shuffle bag
  live_set_id        : int
  set_started_at     : int (ms)

  round_end_delayed   : bool ; round_end_at   : int (ms)
  team_wipe_delayed   : bool ; team_wipe_at   : int (ms)
  base_cost_section   : text              # names the mode's price table
  winning_name        : text

  warmup_deadline : int (ms) ; in_warmup : bool
  spectator_mode  : bool ; spectator_target : GameObject ; ...
```

Invariants:

- **The spawn bookkeeping is four slots wide and the score list is not.** The free-point
  cycle and the last-used point are fixed arrays sized by the respawn-point table's team
  width (four, from [`game_sv_base.h`](game_sv_base.h.md)), while the team *scores* grow
  from configuration. A rebuild must not conflate the two: the first is indexed by the
  authored respawn-point team number, the second by the mode's own team list. They agree
  today only because no mode has more than two of either.
- **The anomaly names and the anomaly identifiers are two parallel lists indexed together.**
  The names come from the level's configuration at round start; the identifiers are filled in
  as zones spawn. A set whose names loaded but whose zones never spawned leaves an empty
  identifier list, which the state sweep treats as the end of the table rather than as a gap.
- **A raw reference to the spectator camera's target is held**, which is why the class takes
  on a destruction notification (below). Every other object reference in the mode is an
  entity identifier resolved on use.

## `game_sv_Deathmatch` — what this layer fixes for its descendants

**Contract** — implemented here and inherited by every competitive mode: the three-phase round
machine (pending, in progress, player scores) and the transitions between them; warm-up as a
round that restarts itself; the money-and-buy economy including the packed loadout encoding;
kill classification and the situational bonus table; respawn protection and its two expiry
rules; respawn-point selection by distance from living enemies; the rotating anomaly sets;
the drop-bag item flow; and the server's own spectator camera.

**Notes** — a descendant that wants none of this has nowhere else to go — capture the artefact
wanted a different round machine and a different buy cycle, and the cost of that decision is
that it re-implements warm-up, the spectator camera, team balancing and the anomaly rotation
from scratch. Compare the two anomaly implementations
([`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) against
[`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md)): same feature,
two codebases, two sets of bugs. A rebuild should lift warm-up, the spectator camera and the
anomaly rotation out of the mode hierarchy entirely — none of them is a rule.

## `RPointData` — the respawn-point scoring record

**Contract** — one candidate respawn point under consideration: its index in the free list,
the distance from it to the nearest living enemy, and a flag marking it temporarily unusable.
Candidates are ordered so that unusable points sort last and, among usable ones, the nearest
enemy sorts first.

```text
RECORD RPointData
  point_index    : int
  min_enemy_dist : real     # squared; 10000 stands for "no enemy anywhere"
  frozen         : bool

ORDER RPointData a BEFORE b
  IF a.frozen AND NOT b.frozen: b comes first
  IF b.frozen AND NOT a.frozen: a comes first
  OTHERWISE a comes first when a.min_enemy_dist < b.min_enemy_dist
```

**Invariants** — the ordering is ascending by danger, and the selector then draws from the
**far end**. Both halves must agree: a rebuild that sorts descending and keeps the same
selector will spawn every player on top of the nearest enemy.

**Notes** — **the frozen dimension is dead.** Nothing anywhere constructs a candidate with the
flag set, so the first two clauses never fire. Some notion of a point temporarily withdrawn
from the cycle was designed and never wired up; artefact hunt later solved the same problem
with a separate administrative block on the point itself, which suggests this was the
abandoned first attempt. A rebuild should implement one of the two, not both.

The distance is kept squared because it is only ever compared, never reported. That is a cost
decision with no behavioural weight, and a rebuild is free to ignore it.

## The overridable rule surface

**Contract** — what a descendant replaces, and what replacing it decides.

```text
# --- Match limits: declared as QUESTIONS, not read as variables ---
get_time_limit() / get_frag_limit() / get_force_respawn()
get_damage_block_limit() / get_warmup_time()
anomalies_enabled() / get_anomalies_time() / damage_block_indicators()

# --- The round machine ---
check_for_round_start() / check_for_round_end()
check_for_time_limit() / check_for_frag_limit()
check_for_anomalies() / check_for_warmup()
on_frag_limit_exceeded() / on_time_limit_exceeded()
on_delayed_round_end(reason) / on_delayed_team_eliminated()
on_team_score(team, minor) / on_teams_in_draw()

# --- Scoring ---
get_kill_result(killer, victim) -> KillResult      # the classification
on_kill_result(result, killer, victim) -> may_bonus
on_give_bonus(result, killer, victim, kind, special, weapon)
processing_victim(victim, killer) / victim_experience(victim)
has_champion() -> bool                             # the overtime rule

# --- Membership and identity ---
check_teams() -> bool          # HERE: false. Team modes answer true.
can_have_friendly_fire() -> bool   # HERE: false. Team modes and CTA answer true.
num_teams() -> int
set_skin(entity, team, index) / on_player_change_skin(client, skin)
load_teams() / load_team_data(section)
load_skins_for_team(section, out) / load_def_items_for_team(section, out)

# --- Placement and damage ---
assign_rp(entity, player) / rp_2_use(entity) -> int   # HERE: always slot 0
check_invincible_players() / check_player_for_invincibility(player)
on_player_hit_player_case(hitter, hitted, hit)        # the damage adjustment
check_for_clear_run(player) / can_charge_free_ammo(section)
fill_death_reject_items(actor, out)

# --- Data ---
anomaly_set_base_name() -> text     # HERE: "deathmatch_game_anomaly_sets"
is_buyable_item(name) -> bool
```

**Invariants** — **every match limit is a method, not a field read.** The round machine never
looks at a setting directly; it asks. That is what lets artefact hunt force its frag limit to
zero and read its own objective count instead, without touching the round machine at all. A
rebuild that inlines the settings into the round checks will have to fork the round machine
per mode.

**`rp_2_use` answering "slot zero" for every entity is the whole of plain deathmatch's spawn
policy**: one shared pool of respawn points, no sides. The team modes override it to return
the actor's team, and that single override is what makes bases defensible.

**Notes** — `check_teams` and `can_have_friendly_fire` ask nearly the same question and answer
it differently across the hierarchy: the multiplayer base says friendly fire is possible and
teams are not; deathmatch says neither; the team modes say both. The pair exists because the
first governs *scoring* (is a kill a team kill?) and the second governs *damage* (does a
bullet from a teammate hurt?), and a mode could in principle want one without the other. None
does.

`on_teams_in_draw` is declared, empty, and called from nowhere in any mode. A draw is a state
the hierarchy can reach — artefact hunt plays on rather than resolving one — but no mode ever
announces it.

## `type_name`

**Contract** — `deathmatch`. The string the session advertises to the server browser and the
value scripts compare against, so it is frozen by the data and by the script surface both.

## Destruction notification

**Contract** — the class opts into being told when any live object is destroyed, so it can
drop its raw reference to the spectator camera's current target.

**Notes** — this is the C++ problem of a dangling pointer, and it survives as the decision
underneath: **the spectator camera holds a live object, not an identifier, and therefore needs
a lifetime contract with the object registry.** A rebuild that stores the target as an entity
identifier and resolves it on use needs no notification at all, which is what the rest of this
file already does everywhere else. The inconsistency is worth removing rather than reproducing.
