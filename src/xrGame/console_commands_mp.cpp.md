# src/xrGame/console_commands_mp.cpp

> The multiplayer console surface: server rules, administration, voting, demo playback and the account commands.

**Needs** — [`console_commands.cpp`](console_commands.cpp.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_mp.h`](game_cl_mp.h.md) · [`game_sv_deathmatch.h`](game_sv_deathmatch.h.md) · [`game_sv_artefacthunt.h`](game_sv_artefacthunt.h.md) · [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`cdkey_ban_list.h`](cdkey_ban_list.h.md) · [`DemoPlay_Control.h`](DemoPlay_Control.h.md) · [`account_manager_console.h`](account_manager_console.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`RegistryFuncs.h`](RegistryFuncs.h.md) · [`date_time.h`](date_time.h.md) · [`xrServerEntities/xrServer_Object_Base.h`](../xrServerEntities/xrServer_Object_Base.h.md) · [`xrEngine/XR_IOConsole.h`](../xrEngine/XR_IOConsole.h.md) · [`xrGameSpy/GameSpy_GP.h`](../xrGameSpy/GameSpy_GP.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: text parsing, live tuning variables and message construction

## Purpose

The multiplayer half of the console surface described in
[`console_commands.cpp`](console_commands.cpp.md). It is a separate file only because of its
size; its registration function is called from the other one's, and a rebuild may merge them.

The content divides into four parts that behave quite differently, and telling them apart is
the point of this page:

1. **Server rules** — the settings that define a match: frag limit, time limit, respawn
   delay, friendly fire, artefact counts. These are *authoritative* and changing one must
   reach every client.
2. **Administration** — kicking, banning, screenshots and configuration dumps of a suspect
   player, map rotation, level and game-type changes. Privileged, and privilege is checked on
   the server, never on the console.
3. **Voting** — a player-facing way to change the rules by consent.
4. **Client settings and account management** — interpolation, demo playback, and the
   matchmaking account commands.

## State

`Stateless` as a module. It references the multiplayer rule variables owned by the game-mode
objects, and keeps one piece of its own: the identifier of the last player printed by the
player listing, so an administrator can act on "that one" without retyping a session number.

## the server-synchronizing setting

**Contract** — most rule settings are registered through a variant that, after storing the
value, tells the server's game state to **re-synchronize**. That broadcast is the whole
difference between a rule and a preference: a frag limit that only the server knows leaves
every client's interface lying.

```text
FUNCTION set_server_rule(value)
  store and clamp as an ordinary bounded setting
  IF a server game state exists THEN signal it to synchronize to all clients
```

**Invariants** — a rule changed mid-match takes effect immediately for the current match.
There is no "applies next round" tier. A rebuild that wants one must add it; the shipped
behaviour is immediate.

**Notes** — which settings use this variant and which do not is not a clean line. Some
genuinely authoritative values (the maximum ping, the corpse limit) are registered as plain
settings and therefore do not synchronize. A rebuild should classify by *whether a client's
behaviour depends on it* and route all of those through the synchronizing form.

## the rule surface

**Contract** — the names below are frozen: server operators set them in configuration files
shipped with the server, and the matchmaking browser reports several of them.

```text
common            forced respawn delay, frag limit, time limit (minutes), warm-up time,
                  damage-block indicators and duration, anomalies enabled and set length,
                  the "hunt the data device" mode, spawn-point freeze time,
                  maximum client ping, reconnect grace, end-of-round hail time
team modes        automatic team balance, automatic team swap, friendly indicators and
                  names, the friendly-fire damage multiplier (0 to 2), the team-kill limit
                  and whether it is punished
artefact hunt     artefact respawn delta, artefact count, artefact stay time,
                  reinforcement time, whether the bearer may sprint, whether bases are
                  shielded, whether carriers are returned to base
capture the       invincibility window after spawning, artefact return time, whether an
artefact          activated artefact returns, score display delay, the divisor relating
                  rank progress to artefact count
spectating        free flight, first person, look-at, free look, team camera -- five
                  independent permissions rather than one mode, so a server can allow
                  exactly the spectator views it wants
voting            enabled (as a bit set, so individual vote kinds can be permitted),
                  whether non-participants count toward the quota, the quota as a
                  fraction, and the vote duration in minutes
networking        server and client update rates, pending-packet limits, the guaranteed-
                  delivery mode, packet logging, compressor enable and statistics, the
                  traffic optimization level
client            interpolation amount, interpolation curve kind (linear, B-spline,
                  Hermite) and its history length; whether to save a demo
```

**Invariants** — the friendly-fire multiplier's range reaching *two* rather than one is
deliberate: some servers punish team damage by doubling it rather than by disabling it.

**Invariants** — the voting-enabled setting is a **bit set**, not a boolean, because the vote
kinds (change map, change game type, kick, restart) are separately permitted. A rebuild
storing it as a flag loses that.

## administration

### kicking and banning

**Contract** — four ban routes, differing in what identifies the target: by product-key
digest with the player connected, by digest alone, by network address, and by display name.
Plus two unban routes (by list index, by address) and two listings (connected players,
banned players). The durations come from a fixed token table rather than a free number, so
an operator picks from a menu of ban lengths.

**Invariants** — **privilege is checked on the server**, by looking up the client that issued
the command and requiring administrator rights. Typing the command is not authority; the
command is refused with an explanatory message when the issuer is not an administrator. This
must not be moved to the console layer, because the same commands arrive from a remote
administrator over the network.

**Notes** — the player listing numbers its rows and remembers the last row it printed, so an
operator can name that player with a fixed word instead of a session number. It is a
convenience with a trap: the remembered identifier goes stale as players come and go.

**Notes** — the ban-by-name command is registered under a name whose implementation was
*switched* to ban-by-digest, with the name-based one left in place unregistered. Banning by
display name is not reachable, which is correct — see
[`cdkey_ban_list.cpp`](cdkey_ban_list.cpp.md) for why a name is not an identity.

### screenshots and configuration dumps

**Contract** — an administrator asks a named client for a screenshot or for the signed
configuration dump described in [`configs_dumper.cpp`](configs_dumper.cpp.md). The request
goes to the server, which relays it to that client and routes the answer back to the
requesting administrator. Two further commands do the same for every connected player at
once, and two settings decide whether the server keeps the returned artefacts on disk.

**Invariants** — the request carries *both* the administrator's identifier and the target's,
because the answer must reach the administrator who asked and not be broadcast.

### map rotation and level changes

**Contract** — three commands over one implementation:

```text
sv_changelevelgametype <level> <version> <game type>   # the general form
sv_changegametype      <game type>       # current level and version implied
sv_changelevel         <level> <version> # current game type implied
```

All three validate before acting: the game type must parse to a known mode, and the
(level, version) pair must appear in the map list **for that game type**. A level that exists
but is not registered for the requested mode is refused, because a mode's maps are authored
for it — a capture-the-artefact map has bases and a deathmatch map does not.

**Invariants** — a level is identified by name **and version**. The same map ships in several
revisions and they are not interchangeable; a rebuild that keys on the name alone will load
the wrong geometry against the right spawn data.

**Notes** — the change is sent as a message to the server rather than performed locally, for
the same reason as every other authoritative action: the level change is a server-wide event
with a handshake, not a local load.

**Notes** — the rotation commands (add a map, list maps, next, previous) operate on the
server's map list, and the anomaly-set command advances the environmental hazard rotation
within the current map. Both are ordinary operator conveniences.

### remote administration

**Contract** — one command with three forms: log in with a user and password, log out, and
send anything else through as a remote command. All three are sent to the server, which
authenticates and executes. Refused outside multiplayer, and the argument is capped in
length before anything is parsed.

**Invariants** — **the password travels to the server in the message**. There is no challenge
and no transport encryption implied at this layer. A rebuild should treat this as a hole to
fix, not a protocol to reproduce; what must be preserved is the *shape* — a session
authenticated once and then issuing commands — not the credential handling.

**Notes** — the command's own persistence is disabled, so the password is never written into
the user's settings file. That is the one part of the handling that is right.

### `sv_status`, `chat`, statistics

**Contract** — print the current server settings (by executing the configuration-listing
command over the server settings group), send a message to every player as the server, and
dump or periodically dump the accumulated match statistics. The chat message is length-capped
before it is sent.

## voting

**Contract** — four commands, split by who may use them: a player starts a vote and casts a
yes or no; only the server stops one. A vote start is refused unless voting is enabled by the
server, no vote is already running, and the match is in progress or pending — a vote during
a warm-up or an end-of-round screen has nothing to act on.

```text
FUNCTION start_vote(proposal)
  IF single player THEN refuse
  IF voting is disabled by the server THEN refuse
  IF a vote is already running THEN refuse         # one at a time
  IF the match phase is not in-progress or pending THEN refuse
  send the proposal to the server
```

**Invariants** — one vote at a time, globally. The quota, the duration and whether
non-participating spectators count toward the denominator are all server settings, so the
same command means different things on different servers.

## demo playback

**Contract** — eight commands over a recorded match: set the playback speed, multiply or
divide it, pause at a given point, cancel a pending pause, rewind until a condition, stop
rewinding, and restart. The pause and rewind commands share an argument parser that reads a
*condition* — a time, a frame, an event — rather than a bare number, which is what makes
"rewind until the round starts" expressible.

**Invariants** — every one of them refuses unless a demo is actually playing. They act on a
playback controller, not on the level.

## the account commands

**Contract** — nine commands wrapping the matchmaking account service: create an account, list
local profiles, log in, log out, delete a profile, print the current profile, suggest
available unique nicknames, register one, and select a profile. These are the console face of
the accounts seam and are the only part of this file that talks to a third-party service.

**Notes** — the whole group is a candidate for removal in a rebuild. The service these address
is discontinued; the commands remain because the menu screens call them. See the accounts
seam for what a replacement owes.

## `name`

**Contract** — change the local player's display name. Refused in single player, refused while
the player is logged in to an account (where the name is the account's), refused if the name
contains a forward slash, and truncated to the account system's nickname length. Preserves
the argument's letter case, unlike most commands. The change is sent as a game event, so the
server is the one that applies it and tells everyone.

**Notes** — the slash is rejected because names appear in paths the statistics writer builds.
That is the only character filtered, and it is a narrower filter than the save-name one in
[`console_commands.cpp`](console_commands.cpp.md) — a rebuild should use the stricter one.

## `g_restart`, `g_restart_fast`, `g_kill`, `g_swapteams`

**Contract** — restart the round (full, or without the between-round screens), kill the local
player, and swap the two teams. The team swap works for both team modes and refuses in the
others, and it ends the current round immediately afterwards, because swapping teams mid-round
would invert everyone's objectives.

**Notes** — the swap temporarily forces the automatic-swap setting on, performs the swap, and
restores the setting. Reusing the automatic path rather than duplicating it is right; doing it
by mutating a global setting is not, and a rebuild should pass the intent as an argument.

## the developer escapes

**Contract** — two settings, absent from shipping builds, that disable the authentication check
and the client/server version match. They exist so a developer can connect a locally built
client to a locally built server. They are registered through the same command type, and the
source notes that the repetition is intentional.

**Invariants** — these must be compiled out of a shipping build, not merely defaulted off. A
server that can be told to skip version checking will be, and mismatched versions produce
divergent worlds rather than honest errors.

## `register_mp_console_commands`

**Contract** — registers every name above. Called from the single-player registration in
[`console_commands.cpp`](console_commands.cpp.md) at bring-up, before any settings file is
read, so that a server's configuration file overrides these defaults rather than the reverse.
