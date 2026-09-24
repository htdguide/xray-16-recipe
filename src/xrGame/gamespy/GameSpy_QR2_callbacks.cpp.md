# src/xrGame/gamespy/GameSpy_QR2_callbacks.cpp

> Answers a directory service's questions about a running match by reading them straight
> off the live server: one property per call, as text, with an empty string standing for
> "this game mode does not have that".

**Needs** — [`xrGameSpyServer.h`](../xrGameSpyServer.h.md) · [`xrGameSpy/GameSpy_Keys.h`](../../xrGameSpy/GameSpy_Keys.h.md) · [`xrGameSpy/GameSpy_QR2.h`](../../xrGameSpy/GameSpy_QR2.h.md) · [`Level.h`](../Level.h.md) · [`game_sv_artefacthunt.h`](../game_sv_artefacthunt.h.md) · [`ui/UIInventoryUtilities.h`](../ui/UIInventoryUtilities.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3. Reads scalars off live objects and appends them to a text buffer. It
is in the game module rather than the matchmaking module only because the properties it
reports are game-mode properties.

## Purpose

This is the *answering* half of server advertisement. A directory service — and, through
it, any player browsing for a game — asks a listed server for named properties; this file
supplies them. It is the only part of the matchmaking integration that reaches into the
game's own state, which is why it lives in the game module and the rest of the integration
lives in [`src/xrGameSpy`](../../xrGameSpy/README.md).

Read it as **a specification of what a server list row contains**, not as vendor glue. A
rebuild that designs a fresh master-server protocol (see the seam — the original service
was shut down in 2014 and these paths now reach nothing) still has to decide what a
browsing player sees, and this is the shipped answer.

## State

Stateless. Every call re-reads the live server. The service hands back an opaque pointer
that the engine set when it registered; it is the running server instance, and the first
act of every callback is to recover it and give up quietly if it is absent or if the
server has no directory endpoint. That "give up quietly" is the file's one universal rule:
**a query is never allowed to fail the match.**

## `callback_keylist`

**Contract** — asked once per query kind (match, player, team), fills a buffer with the
identifiers of every property this server is willing to report. Purely declarative; no
game state is read. The declared set is a compile-time constant, not derived from the
running game mode.

```text
FUNCTION key_list(kind) -> list<PropertyId>
  MATCH kind
    match  -> [ host name, map name, game version, player count, max players,
                uptime, game type, password required, user-password required,
                host port, dedicated, game type id, team count, max ping,
                map rotation, voting enabled, spectator modes,
                frag limit, time limit, damage-block time, damage-block indicator,
                anomalies enabled, anomalies time, warm-up time, forced respawn,
                auto team balance, auto team swap, friendly indicators,
                friendly names, friendly-fire fraction,
                artefact count, artefact stay time, artefact respawn time,
                reinforcement time, shielded bases, return players,
                bearer cannot sprint ]
    player -> [ name, score, deaths, skill, team, spectator, artefacts carried ]
    team   -> [ score ]
```

**Notes** — the match list is the union over *all* game modes, advertised unconditionally.
A deathmatch server still declares that it reports the artefact-respawn time; it simply
answers empty when asked. That trade — a fixed schema with holes, rather than a schema
that varies by mode — is what lets a browser build one table of columns without knowing
the mode first, and a rebuild should keep it.

One property is declared in the enumeration and never reported (an anti-cheat flag), and
one is reported but never declared (round-trip time per player, which the browser asks for
by name anyway). Both are live decisions a rebuild can simply drop.

## `callback_serverkey`

**Contract** — asked for one match property; appends its value to the reply buffer as
text or as an integer. Never fails: an unrecognized property, or a property belonging to a
game mode this match is not running, appends an empty string. Reads the live server and
the live game state; does not lock.

```text
FUNCTION server_property(id) -> text
  server <- the running server, or RETURN on absence
  state  <- server.game_state

  # Four views of the same object, narrowest first. Each is present only if the
  # running mode is that mode or a specialization of it.
  mp     <- state AS multiplayer-base    OR none
  dm     <- state AS deathmatch          OR none
  tdm    <- state AS team-deathmatch     OR none
  ahunt  <- state AS artefact-hunt       OR none

  MATCH id
    host name, map name, game version   -> the server's configured strings
    player count, max players           -> live count, configured cap
    uptime                              -> global clock formatted as day + seconds
    host port, dedicated                -> listening port, whether rendering is off
    password required                   -> 1 IF a join password is set ELSE 0
    user-password required              -> 1 IF the server reserves slots ELSE 0
    max ping                            -> the configured ping above which clients are dropped
    game type, game type id             -> from `state`
    team count                          -> from `mp`
    map rotation, voting, spectator modes,
    frag limit, time limit, damage-block
    time and indicator, anomalies on and
    their period, warm-up time,
    forced respawn                      -> from `dm`
    team balance and swap, friendly
    indicators and names                -> from `tdm`
    friendly-fire fraction              -> from `tdm`, as a percentage: the game holds a
                                           0..1 multiplier, the wire carries 0..100
    artefact count, stay time, respawn
    delta, reinforcement time, shielded
    bases, return players, bearer
    cannot sprint                       -> from `ahunt`
    anything else                       -> empty
  IF the view this property lives on is absent
    RETURN empty                        # not an error: the mode simply has no such rule
```

**Invariants** — every declared property must produce *some* reply, including an empty
one. A property that produces no reply at all desynchronizes the service's field parsing
for the rest of the response, which is why the default branch appends an empty string
rather than asserting.

**Notes** — the "narrow the game state, report empty if it is not that mode" shape is
repeated thirty times through a macro in the original. The macro is incidental; the
decision is that **a match's advertised properties are the union of the rule sets of every
mode, and a mode reports only its own.**

The uptime is formatted as text, not as a count of seconds, and it is derived from the
process-global clock rather than from when the match started — so it reports how long the
*executable* has been running. A rebuild should report match age instead; nothing depends
on the current behaviour.

## `callback_playerkey`

**Contract** — asked for one property of the player at a given index; appends its value.
Silently returns if the index is out of range or that slot has no player record — which,
per the invariant above, means the reply for that field is omitted rather than empty, and
is a real inconsistency with the match path.

```text
FUNCTION player_property(index, id) -> text
  server <- the running server, or RETURN
  IF index >= server.client_count
    RETURN

  # A dedicated server occupies client slot zero itself. Its own slot must not appear
  # in the player list, so every index shifts by one and the bound is rechecked.
  IF server.is_dedicated
    IF index + 1 >= server.client_count
      RETURN
    client <- the (index + 1)-th client in iteration order
  ELSE
    client <- the index-th client in iteration order

  IF client has no player record
    RETURN

  MATCH id
    name       -> player name
    ping       -> last measured round-trip time
    score      -> frags
    deaths     -> deaths
    skill      -> rank
    team       -> team index
    spectator  -> whether the spectator flag is set
    artefacts  -> artefacts carried, BUT ONLY in artefact-hunt or capture-the-artefact;
                  in any other mode nothing at all is appended
    otherwise  -> empty
```

**Notes** — "the *n*-th client in iteration order" is exactly that: the server's client
collection is walked counting until the index is reached. There is no stable player index
and no identifier in the reply, so a browser watching a server sees players appear to
swap rows when anyone leaves. That is a property of the shipped behaviour, not an
accident to preserve; a rebuild with a stable per-player identifier is strictly better.

## `callback_teamkey`

**Contract** — asked for one property of the team at a given index; only the team's score
is answerable. Returns silently if the match is not a deathmatch-derived mode or the index
exceeds the team count.

## `callback_count`

**Contract** — asked how many players or how many teams the reporting loop should iterate
over. Returns the live player count for players. For teams the answer is derived from the
game mode identifier and not from the game state: one for the free-for-all modes, two for
every team mode. An unknown mode is a hard assertion failure — the one place in this file
that does not degrade quietly, because an unknown mode means the mode table and this
switch have diverged.

**Notes** — the team count is hard-coded per mode rather than asked of the mode object,
even though the mode object can answer it and the match-property path *does* ask it. A
rebuild should ask the game state in both places.

## `callback_deny_ip`

**Contract** — asked, before answering a query, whether this source address is banned.
Returns deny if the server's ban list contains the address, allow otherwise. This is the
one callback with a security purpose: it stops a banned player from using the directory
query channel to watch a server they cannot join, and stops the query channel being a
reflection target for an address the admin has already rejected.

## `callback_adderror`

**Contract** — the service could not list this server. Logs the message and hands the
error code to the server, which decides whether to retry, fall back to unlisted operation,
or tell the host. Never throws; listing failure must not stop a match that is already
running.

## `callback_nn` · `callback_cm`

**Contract** — both are empty. Address negotiation for clients behind address translation
is accepted and then ignored, and out-of-band client messages on the directory channel are
dropped. They exist because the service's registration requires every callback in the set
to be present.

**Notes** — the empty negotiation callback is the honest marker of how much of the
matchmaking integration was ever finished: traversal was registered for and never
implemented, so a client behind address translation could be listed and never joined.
A rebuild designing its own master server should treat address negotiation as *the*
feature to get right, since it is the thing a directory buys you that a list of addresses
does not.

A further callback — the service reporting this host's public address, which would have
triggered a challenge-response authentication of the server itself — is present in the
source but disabled. The authentication path it would have driven still exists in the
matchmaking module; nothing calls it.
