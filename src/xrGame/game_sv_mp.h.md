# src/xrGame/game_sv_mp.h

> Declares the multiplayer base every competitive mode derives from, and with it the extension contract: which rules a mode may replace, and which the match layer keeps for itself.

**Needs** — [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_sv_mp_team.h`](game_sv_mp_team.h.md) · [`game_base_kill_type.h`](game_base_kill_type.h.md) · [`game_base_menu_events.h`](game_base_menu_events.h.md) · [`actor_mp_server.h`](actor_mp_server.h.md) · [`cdkey_ban_list.h`](cdkey_ban_list.h.md) · [`ui/UIBuyWndShared.h`](ui/UIBuyWndShared.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`DemoInfo.cpp`](DemoInfo.cpp.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`game_sv_mp_script.cpp`](game_sv_mp_script.cpp.md) · [`game_sv_mp_script.h`](game_sv_mp_script.h.md) · [`xrServer.cpp`](xrServer.cpp.md)
**Tier floor** — T2: a class declaration over session bookkeeping; nothing here touches a device or a frozen memory image

## Purpose

Implemented in [`game_sv_mp.cpp`](game_sv_mp.cpp.md), which carries the algorithms. What this
file decides, and what no other page states, is **the shape of the mode hierarchy**: every
shipped competitive mode is a narrowing of this class, and the set of methods declared
overridable here is exactly the set of rules a mode is allowed to change. Read as an
interface rather than as a declaration, it answers the question a rebuilder actually has —
*what is a game mode permitted to decide, and what is decided for it?*

Four modes descend from it, in two chains:

```text
MultiplayerServer                        # this file: match, respawn, ranks, money, votes, bans
  Deathmatch                             # round machine, economy, spawn placement, anomalies
    TeamDeathmatch                       # two teams, shared score, friendly fire, balancing
      ArtefactHunt                       # one artefact, waves, shielded bases
  CaptureTheArtefact                     # two artefacts, two bases — a SIBLING, not a descendant
```

The second chain is the surprise and it is load-bearing: capture the artefact re-implements
team selection, team balancing, friendly fire, warm-up, the spectator camera and the buy
cycle rather than inheriting them from the team modes. A rebuild that makes it a descendant
of team deathmatch will find half of those rules fighting each other, which is presumably why
the original did not.

**None of this runs in a shipped build.** The transport seam ships its **null** filling by
default on every platform, so a match cannot be started without editing the build description
first (see [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)).
The rules below are complete and were once exercised; a rebuild wanting multiplayer is
finishing a feature rather than restoring one.

## State

The operational record — corpses, ranks, votes, statistics — is in
[`game_sv_mp.cpp`](game_sv_mp.cpp.md). What the declaration alone fixes:

```text
RECORD MultiplayerServerDeclaration
  corpse_list      : queue<int (16-bit)>   # a double-ended queue: entries leave from
                                           # anywhere, not just the front — see Invariants
  ranks            : list<Rank>
  team_list        : list<Team>            # one Team record per playing team, indexed by
                                           # team number; see game_sv_mp_team.h
  item_registry    : handle                # section name <-> small index, shared with the
                                           # buy menu; one instance per session
  spectator_modes  : int (8-bit)           # a bitmask, one bit per permitted camera mode
  cdkey_ban_list   : BanList               # keyed by account digest, not by address

RECORD Rank
  title               : text
  terms               : list<int>   # EXACTLY TWO slots; only the first is read at this level
  bonus_money         : int
  rank_diff_exp_bonus : list<real>  # indexed by the VICTIM's rank

RECORD AmmoRemainder                       # a single (class, count) pair, not a list
  ammo_section : text
  count        : int (16-bit)
```

Invariants:

- **The corpse container must support removal from the middle.** The reap walks the list
  in order and skips a corpse that still owns items, so entries do not leave in arrival
  order. A rebuild using a strict queue will either reap the wrong body or stall on a
  skipped one.
- **A rank's experience thresholds are a fixed pair, and the second is never read here.**
  The width is compiled in at two; artefact hunt is the only mode that reads the second,
  where it means a delivered-artefact requirement. A rebuild should make the extra terms a
  named per-mode field rather than an anonymous second slot.
- **The team list is indexed by the player's team number**, so team numbering and list
  position are the same thing. The modes disagree about whether that numbering starts at
  zero or one, which is the hazard called out in
  [`game_sv_capture_the_artefact_process_event.cpp`](game_sv_capture_the_artefact_process_event.cpp.md).
- **The ammunition remainder is one pair, not a collection.** Only the last partly-consumed
  box of a magazine load survives as an item; earlier boxes are consumed whole.

## `game_sv_mp` — what the match layer owns outright

**Contract** — these are implemented here and a mode does not get to change them: the round
boundary (destroy the world, rebuild from the authored spawn records), the respawn cycle as a
substitution of the client's server object, corpse reaping, the rank ladder and the
experience economy, the money ledger with its itemised bonuses, the vote machinery, the
account-keyed ban list, the map rotation, the statistics dumps, and the rename, chat and
radio-phrase paths. Each is described in [`game_sv_mp.cpp`](game_sv_mp.cpp.md).

**Notes** — the division is not arbitrary. Everything in that list is about *a session*
rather than *a game*: it is the part a dedicated server operator configures and a player
never thinks about. The part a mode replaces is the part a player would describe if asked
what the mode is.

## The overridable rule surface

**Contract** — the declarations that exist to be replaced. Each line is a decision a derived
mode makes; the value given is this layer's answer, which is usually "nothing happens".

```text
# --- Ownership and the veto points ---
on_touch(who, what, forced) -> bool        # DEFAULT: allow everything
on_detach(who, what)                       # DEFAULT: nothing
fill_death_reject_items(actor, out)        # DEFAULT: nothing is rejected on death

# --- Scoring ---
on_player_kill_player(killer, victim, kill_type, special, weapon)   # DEFAULT: nothing
check_teams() -> bool                      # DEFAULT: false — this mode has no teams
can_have_friendly_fire() -> bool           # DEFAULT: true
num_teams() -> int                         # DEFAULT: 0

# --- Menus the client can raise ---
on_player_select_team(packet, sender)      # DEFAULT: nothing
on_player_select_skin(packet, sender)      # DEFAULT: nothing
on_player_buy_spawn(sender)                # DEFAULT: nothing
on_player_open_buy_menu(client)            # DEFAULT: nothing   # valid only while dead
on_player_close_buy_menu(client)           # DEFAULT: nothing

# --- Progression and economy ---
player_rank_up_allowed() -> bool           # a flag the modes bracket around payouts
player_check_rank(player) -> bool          # DEFAULT: the ladder's threshold alone
can_charge_free_ammo(section) -> bool      # DEFAULT: false — nothing is free
```

**Invariants** — **the three veto points are where a mode's rules actually bite.** Touch,
detach and the death-reject list are how a mode says "you may not carry that", "dropping this
is an objective event" and "the weapon in your hands falls where you die". Everything else in
the surface reports or pays; only these three change what is possible.

**The default answers are permissive and empty by design.** A mode that overrides nothing
still runs as a plain arena: everything can be picked up, nothing scores, no menus open.
That makes each mode's own twin readable as a diff against "no rules at all".

**Notes** — the rank-up permission is a *flag with a setter*, not a query, and every mode that
uses it brackets a payout between switching it on and switching it off. The effect is that a
rank change can only land at a moment the mode chose — between lives, or at a delivery —
never in the middle of a firefight. That bracket is the mechanism; a rebuild that makes
rank-up a pure function of experience loses it.

The buy-menu open and close hooks are documented here as valid only from a dead player, and
nothing enforces it at this level. The enforcement is in the modes that implement them, which
is the wrong place for a trust decision about an untrusted message.

## `Rank_Struct` — the progression record

**Contract** — one rung of the ladder: a display title, its experience thresholds, the money
paid on reaching it, and a table of experience multipliers indexed by the *victim's* rank.

**Invariants** — the multiplier table lives on the **attacker's** rung and is indexed by the
**victim's** rung, so the lookup is two-dimensional across one list of records. Beating a
better player pays more; farming a beginner pays less. That table is the entire progression
design and it is pure configuration — no curve is compiled in.

## `SearcherClientByName`

**Contract** — a predicate that finds a connected client by display name. Both the wanted name
and each candidate are folded to lower case before comparison; the wanted name is truncated
into a fixed 512-byte buffer. A client with no player record never matches.

**Notes** — this exists because **kick and ban votes name a player by the text a human typed**,
and the vote must resolve that to a connection before the vote is put. Resolving at proposal
time rather than at execution time is the decision (see the voting section of
[`game_sv_mp.cpp`](game_sv_mp.cpp.md)); this predicate is how.

The fold is ASCII case-folding, not locale-aware — the same rule the resource system uses, and
for the same reason: a locale-aware fold changes which names match.

## Compiled-in rule constants

**Contract** — three numbers are fixed in the declaration rather than read from
configuration.

```text
vote_duration  = 1        # minutes a vote stays open
vote_quota     = 0.51     # fraction of the electorate that must agree
rank_terms     = 2        # experience thresholds per rank, see State
```

**Notes** — the quota is an absolute majority by a hair, and with the default counting rule an
abstainer counts against it (see the voting section of
[`game_sv_mp.cpp`](game_sv_mp.cpp.md)). So in practice a vote needs more than half of
*everyone present*, not half of those who answered.

The first two are only the **initial** values of two console variables the operator can
change at runtime, and the console clamps them: the quota to between zero and one, the
duration to between half a minute and ten minutes. The clamps are the real constraint — a
quota outside that range would make every vote pass or none — and a rebuild should keep them
on whatever surface it exposes the settings through.
