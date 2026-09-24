# src/xrGame/Level_network_start_client.cpp

> The client's connection sequence, cut into six resumable steps so that the loading screen keeps painting while the connection, the level load and the server handshake proceed.

**Needs** — [`Level.h`](Level.h.md) · [`xrServer.h`](xrServer.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`HUDManager.h`](HUDManager.h.md) · [`file_transfer.h`](file_transfer.h.md) · [`physics_game.h`](physics_game.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrEngine/IGame_Persistent.h`](../xrEngine/IGame_Persistent.h.md) · [`xrPhysics/IPHWorld.h`](../xrPhysics/IPHWorld.h.md) · [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: sequencing and subsystem bring-up; the only tier pressure is the polled waits

## Purpose

Starting a session means connecting, agreeing on a level, loading it, bringing up physics
and the network processor, waiting for the server's configuration, and precaching. That is
seconds to minutes of work and it cannot block the frame loop, so it is expressed as a list
of numbered steps the loading sequence calls in order, each returning whether it is done.
A step that returns "not done" is called again next frame.

Single player uses the same sequence. A local *direct connection* replaces the transport
with an in-process link, and the branches marked "direct connect" below are the single-player
path — which is why single player pays for a connection handshake it does not need, and why
the level load is driven by a message from a server that is in the same process.

## State

`Stateless` in itself; every step reads and writes the level's own connection state
(`connected_to_server`, `game_configured`, `deny_m_spawn`, the map sync record, the
accumulated result flag).

## `net_Start_client`

**Contract** — returns false unconditionally. The one-shot entry point was superseded by the
six-step form and is retained only because the declaration is part of the level's surface.
A rebuild should delete it.

## `net_start_client1` — announce

**Contract** — opens the loading screen and puts up "connecting to *name*", where the name is
the first slash-delimited field of the client's option string. Always succeeds.

**Notes** — the option string is a slash-separated list whose first field is the server
address. The double search for a separator is defensive parsing of a string that should
have been a record; a rebuild should parse the options once, at the point they are
composed, and pass a record.

## `net_start_client2` — connect

**Contract** — establishes the connection. On a direct (in-process) connection it creates
the local client on the server side and then **blocks in a spin loop** pumping both the
client's receive queue and the server's update until the connect result arrives. Otherwise
it hands the option string to the transport. Always returns done; success is recorded in
`connected_to_server`, which every later step checks.

**Invariants** — the spin loop only terminates because the server it is pumping is in this
process. It is safe exactly because of that, and must not be reused for a remote
connection.

## `net_start_client3` — identify and load the level

**Contract** — resolves the level name and version to an internal level identifier and loads
that level. On a direct connection it takes the name and version from the server's connect
options; otherwise from the server description the browser supplied, and it rescans the
multiplayer archive path first, because a map may have been downloaded since the last scan.
On an unknown level it disconnects, records the name, version and the server's download
address for the user interface, and reports failure. Blocking: the level load is the long
step.

```text
FUNCTION net_start_client3() -> bool
  IF NOT connected THEN RETURN true            # nothing to do; the failure is already recorded
  IF direct connection THEN
    level_name = the level already named locally
    level_ver  = server connect options' version field
  ELSE
    level_name, level_ver, download_url = the server's description record
    rescan the multiplayer archive search path
  END IF

  level_id = resolve(level_name, level_ver)
  IF level_id is unknown THEN
    disconnect; connected = false
    record name, version and download address; mark the map not loaded
    RETURN false
  END IF
  record name, version, download address; mark the map loaded
  allow spawn messages
  load the level                                # FAIL WITH load failure
  level geometry checksum = 0
  IF multiplayer THEN compute the level geometry checksum
  RETURN true
```

**Notes** — the geometry checksum is computed only for multiplayer, because its only consumer
is the map-verification handshake. It is a whole-file pass over the largest file in the
level and skipping it in single player is worth real seconds.

## `net_start_client4` — physics and the network processor

**Contract** — brings up the physics world and decides where the network processing and
physics run. On a non-direct connection it then **blocks** until the transport reports the
connection established, and blocks again pumping the receive queue until synchronization
completes. Always returns done.

```text
FUNCTION net_start_client4() -> bool
  IF NOT connected THEN RETURN true
  show "spawning"
  create the physics world, threaded or single-threaded per the device flag,
    bound to the level's collision space and object registry
  install the default contact and character-contact shotmark callbacks
  install the per-step time callback
  # The network processor is a frame consumer: it goes on either the worker-thread
  # frame list or the main frame list, never both. Remove from both, then add to one.
  remove the network processor from both frame lists
  IF threaded networking THEN add it to the worker list at high priority
  ELSE add it to the main list at low priority
  IF NOT direct connection THEN
    WHILE the transport has not completed the connection: sleep 5 ms
    WHILE synchronization is not complete: pump the receive queue; sleep 5 ms
  END IF
  RETURN true
```

**Invariants** — the remove-from-both-then-add-to-one shape is required, not tidiness: the
processor may already be registered from a previous session on the other list, and a double
registration means the network is pumped twice per frame.

**Notes** — the contact callbacks are installed *before* any object spawns, because the
physics world calls them from inside its step and an object can be spawned into a contact
on its first frame.

The physics threading and the network threading are independent device flags. Shipping
builds enable both.

## `ClientSendProfileData`

**Contract** — builds an empty player-state record, exports it and sends it to the server over
the encrypted channel, reliably and in order. Called in response to the server's
authentication challenge.

**Notes** — the record sent is default-constructed, so the content is whatever the player
state's export defines as its initial values. The server uses it as the shape of the record
it will maintain for this client, not as data.

## `net_start_client5` — resources

**Contract** — uploads the deferred texture set to the graphics device and verifies it, unless
this process is a dedicated server. Then arms the two gates the synchronization step needs:
the connection-data request is marked unsent, and spawn messages are denied.

**Invariants** — `deny_m_spawn` is set here and cleared in `synchronize_map_data`. The window
between them is exactly the period in which the level exists but the game rules do not, and
a spawn arriving in it has nowhere to register.

## `net_start_client6` — synchronize and finish

**Contract** — the only step that can return "not done". Runs the map-verification and
configuration gates; if either is still pending, returns false to be called again next frame.
Once the rules are configured it loads the head-up display, notifies it and the game rules of
the connection, creates the file-transfer client for multiplayer, and precaches sixty frames
before handing control to the running game.

```text
FUNCTION net_start_client6() -> bool
  IF NOT connected THEN
    overall result = failure
    close the loading screen
    RETURN true
  END IF

  IF NOT synchronize_map_data() THEN RETURN false     # call me again next frame

  IF NOT game rules configured THEN
    close the loading screen
    RETURN true                # done, but without success: the caller sees the flag
  END IF

  IF this process renders THEN load the head-up display and tell it we connected
  IF game rules exist THEN
    rules.on_connected()
    IF multiplayer THEN create the file-transfer client
  END IF
  show "synchronising"
  precache 60 frames
  overall result = success
  close the loading screen
  RETURN true
```

**Notes** — the sixty precache frames are the engine rendering the scene without presenting
it, so that every shader is compiled, every texture resident and every buffer uploaded
before the player's first visible frame. Skipping it produces a several-second hitch a few
seconds into play instead. The count is a budget, not a measurement: sixty frames is enough
to have touched everything visible from a spawn point on the shipped levels.

The file-transfer client exists only in multiplayer because its only jobs — downloading a
map the client lacks, uploading a screenshot on demand — are server-driven.

## `rescan_mp_archives`

**Contract** — rescans the multiplayer archive search path so that an archive downloaded
during this session becomes visible to the virtual filesystem. No-op if the path is not
configured.
