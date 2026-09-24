# src/xrGameSpy/GameSpy_QR2.cpp

> The server's side of matchmaking: register with the master list, keep a heartbeat alive,
> and answer field-by-field queries about the running match. Also the process-wide table
> that binds a field id to the name that travels.

**Needs** — [`GameSpy_QR2.h`](GameSpy_QR2.h.md) · [`GameSpy_Keys.h`](GameSpy_Keys.h.md) · [`xrGameSpy_MainDefs.h`](xrGameSpy_MainDefs.h.md) · [Seam: Multiplayer matchmaking and accounts](../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`GameSpy_QR2.h`](GameSpy_QR2.h.md)
**Tier floor** — T2. It owns a listening socket's lifetime and a callback surface invoked
from the query engine; nothing needs manual layout.

## Purpose

A server that wants to be found must do two things: tell a master list it exists and keep
telling it, and answer a direct query from a browsing client. Both are this file's job.
It does not hold the answers — the game does, through callbacks — it holds the
*registration*: which port, which title, which fields exist and what they are called.

The field-name registration is process-wide and outlives any one server instance, which
is why the client-side list object also constructs one of these (see
[`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md)) purely to read the table back.

## State

```text
# The callbacks a server must supply. This is the contract a rebuild's game layer has to
# satisfy; the wire format underneath is replaceable, this shape is not.
RECORD AdvertiserCallbacks
  on_server_field  : (field_id, out_buffer) -> void          # append this server's value
  on_player_field  : (field_id, row, out_buffer) -> void     # append player `row`'s value
  on_team_field    : (field_id, row, out_buffer) -> void     # append team `row`'s value
  on_field_list    : (which : {server|player|team}, out_keys) -> void
                                                             # declare which fields exist
  on_row_count     : (which : {player|team}) -> int          # how many rows exist
  on_register_error : (error, message) -> void               # master-list registration failed
  on_nat_request   : (cookie) -> void                        # a client wants NAT negotiation
  on_client_message : (bytes) -> void                        # game-specific message via master
  on_deny_address  : (sender_address) -> (deny : bool)       # refuse to answer this address
  owner            : ServerHandle                            # passed back to every callback
```

`Stateless` otherwise — the object holds no fields. The query engine holds the socket and
the registration; this type is a set of free functions wearing a class.

## `Init`

**Contract** — binds a query socket, registers the server with the master list under this
build's title identity, installs the callback set, and registers this game's own field
names. Returns whether it succeeded; a failure leaves the server running but invisible to
the master list and unqueryable. Does not block on the master list's answer — a
registration failure arrives later, through `on_register_error`.

```text
FUNCTION init(port : int, publicly_listed : bool, cb : AdvertiserCallbacks) -> bool
  IF port == -1
    port <- advertise_base_port          # 5445
  ELSE
    port <- clamp(port, port_range_min, port_range_max)

  err <- start_advertiser(bind_address = any,
                          port,
                          title.short_name, title.secret,
                          publicly_listed,
                          nat_negotiate = false,          # see the note
                          cb.on_server_field, cb.on_player_field, cb.on_team_field,
                          cb.on_field_list, cb.on_row_count, cb.on_register_error,
                          cb.owner)
  IF err THEN RETURN false

  register_field_names()
  install cb.on_client_message
  install cb.on_nat_request
  install cb.on_deny_address
  RETURN true
```

**Invariants**

- A port of `-1` means *use the default*; any other value is clamped into the legal port
  range rather than rejected. A server configured with a nonsense port therefore
  advertises on port 0 or 65535 and fails quietly, which is worse than refusing. A
  rebuild should reject.
- The *publicly listed* flag is the server's `public` configuration option, and it does
  double duty: the same option decides whether the server authenticates clients' product
  keys (see [`GameSpy_GCD_Server.cpp`](GameSpy_GCD_Server.cpp.md)). A private server is
  therefore also an unauthenticated one. That coupling is not obviously intended and a
  rebuild should separate the two.
- Field names are registered *twice* in practice: once here and once by the client-side
  list object's construction. Registration is idempotent by id, so this is harmless, but
  it confirms the table is process-global, not per-server.

**Notes**

**NAT negotiation is requested as disabled.** The flag passed to the advertiser is a
literal off, and the negotiation callback the server installs does nothing. So: the game
*asks the matchmaking layer for NAT traversal* — the capability is present in the
interface, the client asks whether a server needs it (see `CheckDirectConnection` in
[`GameSpy_Browser.cpp`](GameSpy_Browser.cpp.md)), and the server has a place for the
negotiation request to arrive — but the shipped engine declines it at both ends. Stated
as a capability, what the game needs is: *a client that cannot reach a listed server's
advertised address directly must be able to obtain a path to it, mediated by the
matchmaking service, at the point of connection.* The shipped engine needs nothing else
from NAT: there is no hole-punching state machine here, no relay fallback, and no address
rewriting. A rebuild can satisfy this with nothing at all and lose exactly what the
original loses — servers behind NAT are unreachable.

**Could not recover** — why negotiation was disabled. There is no comment and no
configuration path that turns it on.

## `RegisterAdditionalKeys`

**Contract** — binds each of this game's field ids to the text name that travels on the
wire. Idempotent; global to the process. Must run before any query is answered, which is
why both the server's init and the client list's construction call it.

The bindings themselves are the table in [`GameSpy_Keys.h`](GameSpy_Keys.h.md), which is
where a rebuilder should read them: this function is that table expressed as a sequence of
calls, and holding it as data instead is strictly better.

**Notes** — five commented-out per-player registrations (name, frags, deaths, rank, team)
record the moment it was realised the standard per-player fields already carry them. Their
ids are burned; see the invariant in [`GameSpy_Keys.h`](GameSpy_Keys.h.md).

## `Think`

**Contract** — one poll per server frame. Sends the heartbeat when it is due, and reads
and answers any pending query. Does not block. The server calls this only while its
advertiser is initialised.

**Notes** — the heartbeat interval is the query engine's, not this engine's, and nothing
here configures it. What the game needs stated as a capability: *a listed server must
refresh its listing periodically, and a server that stops refreshing must fall off the
list.* A rebuild's master-server protocol owns the interval.

## `ShutDown`

**Contract** — deregisters the server and closes the query socket. Called from the
server's destructor. Does not wait for the master list to acknowledge.

## `BufferAdd` · `BufferAdd_Int` · `KeyBufferAdd`

**Contract** — append a text value, an integer value, or a field id to the buffer the
query engine handed to a callback. These are the only way a callback answers. Each is a
one-line forward.

**Notes** — values go out as text either way; the integer form exists so the game layer
does not format numbers itself. A rebuild with a typed advertisement can drop the
distinction, but must then decide a type for every field in the catalogue, which the
original never had to.

## `RegisteredKey`

**Contract** — the text name bound to a field id. Reads the process-global table directly,
with no bounds check: an unregistered id yields whatever the table holds. Used by the
client to turn a field id into the name it reads a value by.

## `GetGameVersion`

**Contract** — the build's version string, so the server can answer the standard version
field without reaching outside this module. Duplicates
[`xrGameSpy.cpp`](xrGameSpy.cpp.md); a rebuild keeps one.
