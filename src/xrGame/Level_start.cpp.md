# src/xrGame/Level_start.cpp

> The level's bring-up sequence — a chain of resumable steps that stands up a server, connects a client to it, and reports the game as ready or diagnoses why it is not.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`MainMenu.h`](MainMenu.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`UIGameCustom.h`](UIGameCustom.h.md) · [`ui/UICDkey.h`](ui/UICDkey.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: the sequence is ordinary control flow, but it must yield to the frame loop between steps so the loading screen keeps drawing.

## Purpose

Starting a level is slow — seconds of disk, several network round trips, a texture
upload — and it must not block the frame loop, because the loading screen is drawn by
that loop. This file therefore expresses bring-up as a **queue of loading steps**: each
step does one bounded piece of work and returns whether it is finished; the frame loop
pops one step per frame and redraws in between. A step that is not finished stays at the
head of the queue and is retried next frame, which is how the network waits are written
without a blocking loop.

Single player and multiplayer run the *same* sequence. Single player is the case where the
server is created in this process and the client connects to it directly, with the
network transport bypassed; the vocabulary of "server options" and "client options"
survives everywhere regardless.

## State

```text
RECORD LevelStartup
  server_options  : text     # level name / game type / tuning, as one slash-separated string
  client_options  : text     # player name, port, password, key — same encoding
  start_succeeded : bool     # AND-ed across every step; one failure poisons the rest
  connect_error   : enum     # why the server refused, if it did
```

The two option strings are the engine's universal way of naming a session: they arrive
from the console, the main menu, a demo file's header or a server's level-change message,
and every one of those producers writes the same syntax. A rebuild may use a structured
record instead, but must keep a *textual* form because the demo header stores one
verbatim.

**Invariant** — `client_options` always carries a player name by the time the connection
is attempted. If none was supplied, one is taken from the stored profile, then from the
operating system's user or machine name. An empty name is a programming error, not a
recoverable case.

## `start_session`

**Contract** — Begins bring-up for the given server and client option strings. Does not
complete it: it normalizes the options, decides whether this session is being recorded,
and enqueues the six startup steps. Returns the running success flag, which at this point
is still optimistic.

```text
FUNCTION start_session(server_options, client_options) -> bool
  start_succeeded = true
  begin loading screen

  name = stored player name, else OS user name, else machine name
  client_options = client_options with a name field, inserted or repaired
      # "repaired" means: a name key present but empty gets the resolved name spliced in
      # while any trailing fields after it are preserved

  IF not replaying a demo AND this is not a single-player session AND recording is on
    begin recording this session to a demo file

  enqueue steps: start1 .. start6
  RETURN start_succeeded
```

## The six server-side steps

**Contract** — Each step runs on one frame, checks the running success flag first, and
becomes a no-op once bring-up has already failed. The order is load-bearing and is the
reason they are six named steps rather than one function.

```text
STEP start1   # decide what kind of server, and that the level exists
  IF server_options is empty            # pure client: skip, a server exists elsewhere
    RETURN done
  create server: the plain one for single player, the matchmaking-aware one otherwise
  IF this session is not an alife session
    resolve (level name, level version) from the options
    IF the level is not in the installed level list
      start_succeeded = false           # every later step now short-circuits
  RETURN done

STEP start2   # bring the server up and let it choose the final level name
  connect the server to its own options; on refusal record the error and fail
  server.build_default_state()
  level name = the name the server settled on   # it may differ from the requested one

STEP start3   # make the client options self-sufficient
  IF the client has no port, copy the server's listening port in
  IF the server has a password and the client does not, copy it in
  IF the client options carry a key, hand it to the console so it reaches the key store

STEP start4   # splice the client sub-sequence in front of the rest
  remove self from the queue
  push the six client steps onto the FRONT of the queue
  RETURN not-done                       # yield this frame; client steps run first

STEP start5   # announce readiness
  send the local player's exported state as a "client ready" message
  IF this process is a client of a remote server but also holds a local server
    clear that local server's world                # it is a leftover, not authoritative

STEP start6   # finish or explain the failure
  reset and reload the bullet manager's tuning
  end loading screen
  IF start_succeeded
    run any deferred console command supplied on the command line
    tell the game UI it is connected
  ELSE
    diagnose (below) and return to the main menu
```

**Notes** — Step 4 exists purely to interleave: the client half of bring-up must happen
*between* step 3 and step 5, and expressing that by re-ordering a queue is how the
original avoids nesting the two sequences. A rebuild with coroutines writes this as one
linear routine and deletes the step names; the ordering constraint is what must survive.

The "clear the local server" in step 5 catches the case where a player who was hosting
joins someone else's game: the stale local world would otherwise keep updating.

## Failure diagnosis

**Contract** — When bring-up failed, the level tears itself down, returns to the main
menu, and picks one of four explanations. The distinction matters because three of them
are actionable by the player and one is not.

```text
IF the server refused the connection AND this is a real network session
  -> "different version / rejected": offer the multiplayer browser
ELSE IF the level was never loaded AND a level name was known AND the server accepted us
  -> "you do not have this map": show its name, version and download address
ELSE IF the level loaded but its geometry checksum disagreed with the server's
  -> "your copy of this map is corrupted": same dialog, same download address
ELSE
  -> silent return to the main menu
```

**Notes** — The level destroys itself from inside one of its own methods; the original
flags this as a hazard at every one of the four sites. A rebuild must make teardown the
caller's job — return a verdict, let the owner destroy — because the alternative is a
lifetime rule that cannot be checked.

## `configure_client_game`

**Contract** — Handles the server's "here is the game mode" message. Reads a game-mode
name, and if the current mode already has that name, returns without touching anything —
a reconfiguration mid-session must not discard live mode state. Otherwise it destroys the
old mode object, instantiates the one the class registry maps that name to, initializes
it, enables update compression for non-single-player sessions, and runs the game-specific
half of level loading.

**Invariants** — After this returns, the client's game mode exists and its name matches
the server's. The game-specific load
([`Level_load.cpp`](Level_load.cpp.md)) must run *after* the mode exists, because static
particles are filtered by game type.

## `open_demo`

**Contract** — Opens a recorded session file, reads its header and returns the server
option string stored in it, so the caller can start that exact session again. Playback is
then started by feeding those options to `start_session` against a loopback address — a
recorded session replays as a session whose packets come from a file instead of a socket.
See [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md).
