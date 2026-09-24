# src/xrGame/game_sv_artefacthunt.h

> Declares artefact hunt as a narrowing of team deathmatch: one objective with a four-state life, seven match tunables, and — the interesting part — three inherited rules switched off by empty overrides.

**Needs** — [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_artefacthunt_process_event.cpp`](game_sv_artefacthunt_process_event.cpp.md) · [`game_sv_teamdeathmatch.h`](game_sv_teamdeathmatch.h.md)
**Used by** — [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`game_sv_artefacthunt_process_event.cpp`](game_sv_artefacthunt_process_event.cpp.md) · [`GameSpy_QR2_callbacks.cpp`](gamespy/GameSpy_QR2_callbacks.cpp.md) · [`xrServer_Connect.cpp`](xrServer_Connect.cpp.md)
**Tier floor** — T2: a class declaration over session rules

## Purpose

The rules are in [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md). Three things are
this declaration's own and appear nowhere else.

1. **The objective's state machine**, as an enumeration with four values and a fixed set of
   legal transitions. The implementation moves between them at half a dozen sites; only here
   are they visible as one closed set.
2. **The match tunables as overridable accessors** rather than direct reads — the same
   indirection [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) uses, extended with this
   mode's own seven settings.
3. **Three inherited rules disabled by declaring them empty.** Each is a real design
   decision, invisible in the implementation file because there is nothing there to see.

## State

The operational record is in [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md). The
declaration adds two facts about it:

- **The mode carries its own random stream** for choosing artefact spawn points, separate
  from the shared one every other draw uses. Nothing in the source says why. The commented-out
  code it replaced kept a bag of unused points and excluded the last one, so the *intent* was
  a non-repeating cycle; the live code is a plain uniform draw that can land on the same point
  twice. A rebuild wanting the original behaviour reinstates the bag and then has no use for a
  separate stream at all.
- **`artefact_brought_to_base` is write-only.** It is set to true in two places during
  creation and read nowhere in the codebase. It is not the flag the delivery path uses — that
  one is the drop-suppression flag. What it was meant to gate is not recoverable.

## `ARTEFACT_STATE` — the objective's life

**Contract** — the artefact is in exactly one of four states, and each state says which of the
two deadlines is armed.

```text
ENUM ArtefactState
  NONE            # before the first round; no deadline
  NO_ARTEFACT     # none exists; the SPAWN deadline is running
  ON_FIELD        # it exists and is loose; the REMOVAL deadline is running
  IN_POSSESSION   # somebody carries it; neither deadline runs

TRANSITIONS
  NONE          -> NO_ARTEFACT     on creation, and on every re-arm
  NO_ARTEFACT   -> ON_FIELD        the spawn deadline expired and a contest exists
  ON_FIELD      -> IN_POSSESSION   an actor picked it up
  IN_POSSESSION -> ON_FIELD        the carrier dropped it, or died
  ON_FIELD      -> NO_ARTEFACT     the removal deadline expired; it despawned
  IN_POSSESSION -> NO_ARTEFACT     it was delivered, and consumed
  any           -> NO_ARTEFACT     the recovery check found the entity gone
```

**Invariants** — **exactly one deadline is armed at a time**, and the state says which. The two
arming routines enforce it by each clearing the other's deadline. A rebuild that leaves both
armed spawns a second artefact while the first is still on the field, and the mode then has
two objectives and one score.

**The state and the artefact's entity identifier can disagree**, which is why a recovery check
exists at all. The identifier is the authority — every predicate tests it first — and the
state is a summary. A rebuild with one owner for the objective's lifetime can collapse the
two, but must keep the guarantee the recovery check provides: the objective may never be
permanently missing.

**Notes** — the enumeration's first value collides with a name the platform headers define,
and the declaration undoes that definition to make room. That is the ordinary hazard of
sharing a global name space with an operating system's headers; the decision underneath is
only that the "nothing has happened yet" state needs a name, and a rebuild should give it a
less generic one.

## The match tunables

**Contract** — seven settings, each reachable only through a method so a descendant could
replace the policy rather than the value. All seven are also set from the server's option
string at session creation.

```text
artefacts_count()          -> int    # deliveries to win; also the client's "frag limit"
artefacts_respawn_delta()  -> int    # SECONDS from a loose artefact leaving to the next spawn
artefacts_stay_time()      -> int    # MINUTES a loose artefact survives; 0 = forever
reinforcement_time()       -> int    # SECONDS between waves
                                     #   0  = respawn immediately, no waves
                                     #  -1  = respawn only when the artefact is delivered
shielded_bases()           -> bool   # a player on his own base takes no damage
return_players_to_bases()  -> bool   # a delivery teleports the survivors home
bearer_cannot_sprint()     -> bool
```

**Invariants** — **the reinforcement setting is three-valued and two of the three are
sentinels.** A positive number is an interval; zero disables waves entirely; minus one
replaces the wave with the delivery. Those are three different modes of play selected by one
integer, and the third of them additionally enables team elimination as a win condition. A
rebuild should make it an explicit choice with an optional interval; the sentinel packing is
the source of the awkward tests scattered through the implementation.

Two adjacent settings are in **different units** — the respawn gap in seconds, the stay time
in minutes — and nothing but the conversion factors says so. Normalise at the configuration
boundary.

**Notes** — the delivery count doubles as the frag limit the client displays, which is why
artefact hunt must force the inherited frag limit to zero: two different numbers would
otherwise compete to end the round.

## Rules switched off

**Contract** — three inherited behaviours are disabled by declaring them as doing nothing.
Each is a decision.

```text
on_player_fire(client, packet)        -> nothing
    # Deathmatch: firing forfeits respawn protection immediately.
    # Here: protection is the BASE SHIELD, not a respawn timer. A defender must be
    # able to shoot from his own base without losing it — that is what a base is for.

victim_experience(victim)             -> nothing
    # Deathmatch: a player is promoted at the moment he dies, so a rank change
    # (which alters his loadout) lands between lives.
    # Here: promotion happens only inside the delivery payout's rank-up bracket,
    # so rank tracks OBJECTIVE play rather than survival.

update_team_score(killer, old_kills)  -> nothing
    # Team deathmatch: a player's frag delta rolls into his team's score.
    # Here: the team score counts DELIVERIES AND ELIMINATIONS ONLY. Kills pay money
    # and experience and never move the scoreboard.
```

**Invariants** — the third is the mode's identity. A team can be losing the firefight
comprehensively and still win, because nothing about a kill reaches the score. A rebuild that
leaves the inherited scoring in place has built team deathmatch with a prop in it.

**Notes** — disabling a rule by overriding it to do nothing leaves no trace in the
implementation file, which is why these are easy to miss and why they are stated here. A
rebuild expressing modes as composed rule sets rather than as an inheritance chain gets this
for free: the mode simply does not include the rule.

## The overridable rule surface

**Contract** — what artefact hunt replaces on top of team deathmatch, beyond the objective
itself.

```text
assign_rp(entity, player) / rp_2_use(entity)    # per-team points; kill the occupant if full
set_rp(entity, point) / check_rp_unblock()      # the one-second spawn reservation
check_player_for_invincibility(player)          # the base shield
on_player_hit_player_case(...)                  # the shield's damage nullification
get_kill_result / on_kill_result / on_give_bonus  # bearer kills promoted to CRITICAL
player_check_rank(player)                       # deliveries gate promotion
on_player_ready(client)                         # the wave gate
on_player_buy_spawn(client)                     # paying to skip the wave
on_object_enter_team_base / on_object_leave_team_base
on_touch / on_detach / on_create                # the artefact's ownership transitions
check_for_clear_run(player)
on_time_limit_exceeded()                        # duplicates the mode's own time-limit check
anomaly_set_base_name() -> "artefacthunt_game_anomaly_sets"
can_have_friendly_fire() -> true
load_teams()                                    # neutral plus two playing teams
```

**Notes** — **the time-limit hook and the mode's own round-end check are two paths to the same
decision**, they disagree on a tie, and which one runs first is an ordering accident. Named
here because the declaration is where the duplication is visible: the mode overrides the
inherited hook *and* writes its own check. A rebuild should have exactly one.

## `CheckForAnyAlivePlayer`

**Contract** — private, and run only in the delivery-driven reinforcement mode while an
artefact exists. If no connected, playing, non-skipped player is alive anywhere, everybody is
respawned.

**Invariants** — this is the mode's **deadlock breaker**. With reinforcement tied to delivery,
nothing brings the dead back except somebody scoring — and nobody can score if everybody is
dead. The elimination check catches the case where *one* team is wiped; this catches the case
where both are, which the elimination check cannot see because neither team is the winner.

**Notes** — it is guarded on an artefact existing, so a simultaneous wipe during the gap
between artefacts is not caught here. The spawn re-arm covers it a moment later. A rebuild
should test the condition unconditionally; the guard buys nothing.

## `type_name`

**Contract** — `artefacthunt`. Advertised to the server browser and compared against by
scripts, so frozen as text.
