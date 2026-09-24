# src/utils/mp_gpprof_server/gamespy_sake.cpp

> Logs in to the vendor's account service with a hard-coded service account, then queries its record store for a batch of players' award and streak totals.

**Needs** — [`gamespy_sake.h`](gamespy_sake.h.md) · [`profile_data_types.h`](profile_data_types.h.md) · [`atlas_stalkercoppc_v1.h`](atlas_stalkercoppc_v1.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)

**Used by** — reached through its declarations in [`gamespy_sake.h`](gamespy_sake.h.md); callers name that, not this file.

**Tier floor** — T1 as written: it fills a fixed-layout request record, hands the vendor library pointers into text it continues to own, and uses a fixed-width secret buffer. The protocol shape underneath is T2.

## Purpose

**This file reaches nothing today.** The vendor's service was shut down in 2014
([Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts));
every call in it now fails at the first availability check, and the process exits during
start-up. It is recipe'd for what it documents rather than for what it does: **the shape
of the request a profile service must answer, and the shape of the response the rest of
this tool consumes.** A rebuild designing a replacement service reads this page to learn
what to serve, and writes none of the vendor mechanics.

The exchange, stripped of the vendor:

1. **Prove the service exists** before doing anything, by name.
2. **Log in as a service account** — a fixed identity belonging to the tool, not to any
   player — and obtain a session ticket.
3. **Open the record store** with the game's numeric identity and a shared secret, and
   present the ticket.
4. **Query one table** — the profile table named in
   [`profile_data_types.cpp`](profile_data_types.cpp.md) — for the sixty-seven columns the
   profile record needs, filtered to a set of player names, paged.
5. **Decode each returned row** into a profile, keyed by the player-name column.

## State

```text
RECORD Session
  identity       : ServiceAccount        # fixed; see Invariants
  ticket         : bytes                 # returned by login, presented on every query
  profile_id     : int                   # the service account's own id
  columns        : list<text>            # the 67 field names, built once
  pending        : bool                  # a query is outstanding
  offset         : int                   # paging cursor into the result set
  requested      : list<text>            # the batch's names, deduplicated
  results        : map<text, ProfileData>

RECORD GameIdentity                      # constants of the registration
  name        : text                     # the game's short name
  id          : int
  product_id  : int
  namespace_id: int
  secret      : bytes (6 significant, in a 32-byte buffer)
```

**Invariants**

- **The service account's credentials are compiled in**, as is the record store's shared
  secret. Both are in the source in plain text. That was survivable because the account
  could only *read* profiles, but it is still a credential in a shipped artefact, and a
  rebuild must take them from configuration instead.
- **The secret is written into a fixed-width buffer that is zeroed first and then filled
  only in its first six bytes.** The remaining bytes being zero is part of what is sent;
  the buffer width is the vendor's. A rebuild sending only the six significant bytes sends
  something different.
- **The session is a singleton and asserts it.** The vendor library keeps process-global
  state; two sessions corrupt each other.
- **The column list is built once**, in a fixed order: the seven streak columns first,
  then each award's count and date column in award order, then the player-name column
  last. The decode does not depend on that order — it matches by name — but the *count*
  does: the request's field count is baked into the record's size.
- **The column names are pointers into the registry's own literals**, never copies. That
  is why the registry's lookups return long-lived text and why the request may hold them
  past the call that built it.
- **The filter is a text expression the service parses**, built by joining one equality
  clause per requested name with a disjunction operator. It is string concatenation into a
  query language, and the only defence against injection is dropping any name containing a
  quote — which is done here, at the point of use, rather than where the name entered.
  A rebuild should reject at the door and parameterize the query.
- **The batch is paged at a fixed maximum rows per response**, and the continuation
  condition is "the response was full *and* the cursor has not reached the requested
  count". A batch larger than the page yields several round trips; one whose names mostly
  do not exist terminates early because the response is short.

## `sake_processor` — establishing the session

**Contract** — checks the service's availability by game name, blocking and polling until
it answers; initializes the account layer; logs in with the fixed service account and waits
for the result; then opens the record store with the game identity and secret and presents
the ticket. Fails hard at each step, with a distinct message. Blocks throughout. Must be
constructed exactly once per process.

```text
FUNCTION establish_session()
  WHILE availability_of(game.name) IS pending
    wait briefly
  FAIL WITH service_unavailable IF NOT available     # this is where it stops, today

  initialize_account_layer(game.product_id, game.namespace_id)
  login(service_account.nick, service_account.email, service_account.password)
      -> on completion: record profile_id and fetch the session ticket
  FAIL WITH login_failed IF login did not succeed

  open_record_store()
  bind_game(game.name, game.id, game.secret)
  present_ticket(profile_id, ticket)
  build_column_list()
```

**Notes**

- The availability check is not ceremony: the vendor operated regional endpoints and this
  is what resolved them. A replacement service needs *some* equivalent — a reachable
  endpoint discovered before the first query — or the tool's first user request is what
  discovers the outage.
- The login result arrives through a completion callback even though the call is made in
  blocking mode, so the ticket is captured in the callback and read afterwards. That is
  the vendor's shape; a rebuild returns it.

## `think`

**Contract** — give the session a time slice, in milliseconds, to advance its outstanding
asynchronous work. **Nothing progresses between calls**: completion callbacks fire only
inside one. This is why the request pump in
[`requests_processor.cpp`](requests_processor.cpp.md) is a polling loop rather than an
event wait, and it is the single most consequential property of the vendor library on this
tool's architecture. A rebuild with a normally asynchronous client deletes the pump.

## `begin_fetch`, `add_name`, `fetch`

**Contract** — `begin_fetch` clears the batch and the previous results and rewinds the
paging cursor. `add_name` appends a name, ignoring one already in the batch. `fetch`
builds the filter, issues the query and marks a request outstanding; a query that could
not even be started is failed immediately through the same completion path, so the caller
never waits forever.

```text
FUNCTION fetch()
  request.table      <- the profile table
  request.columns    <- columns
  request.use_cache  <- true
  IF build_filter() IS empty THEN RETURN          # every name was rejected; see Notes
  request.filter     <- the filter text
  request.offset     <- offset
  request.max_rows   <- the page size

  handle <- issue(request, on_complete = decode_response)
  pending <- true
  IF handle IS none
    report the start failure
    decode_response(no input, no output)          # completes the batch as empty
```

**Invariants**

- **The duplicate check in `add_name` is a linear scan of the batch.** With a batch the
  size of a server's player list that is fine; it is quadratic and a rebuild uses a set.
- **A batch whose names are all rejected returns with `pending` still set**, because the
  early return happens after the flag would have been cleared but before a query exists.
  The pump then waits for a completion that never comes. That is a real hang, reachable by
  a single request for a name containing a quote. A rebuild must complete the batch on
  this path.
- The request asks the service to use its own cache. The tool caches too
  ([`profiles_cache.cpp`](profiles_cache.cpp.md)); the two are independent and the remote
  one has unknown semantics.

## `request_callback` — completing and paging

**Contract** — decodes whatever rows came back, advances the cursor, and either issues the
next page or marks the batch done. Runs on the session's own thread, inside `think`.

```text
FUNCTION decode_response(input, output)
  IF output EXISTS
    FOR EACH row IN output.rows
      profile, name <- decode_row(input.columns, row)
      IF name IS NOT empty THEN results[name] <- profile
    offset <- offset + output.row_count
    IF output.row_count == page_size AND offset < requested.count
      fetch()                                     # next page; stays pending
      RETURN
  pending <- false
```

**Notes**

- The paging test compares the cursor against the *number of names asked for*, which is
  only an upper bound on the number of rows: a player with no record returns no row, so a
  full page followed by a short one terminates correctly, but a full page whose names all
  existed and whose count happens to equal the batch size stops one page early. It is a
  bound, not a count, and a rebuild should page on the service's own "more rows" signal.

## `process_record` — decoding one row

**Contract** — turns one returned row into a profile and the player name it belongs to.
Reports failure when the row carried no name, which is how a malformed row is dropped
rather than filed under an empty key.

```text
FUNCTION decode_row(requested_columns, row) -> (ProfileData, text)
  FOR EACH column IN requested_columns
    field <- the row's field with this column's name       # see Notes on the search
    IF field IS none THEN CONTINUE

    IF column NAMES an award's count column
      profile.awards[that award].count <- field.as_short
    ELSE IF column NAMES an award's date column
      profile.awards[that award].last_reward_date <- field.as_int
    ELSE IF column NAMES a streak column
      profile.best_scores[that streak] <- field.as_int
    ELSE IF column IS the name column
      name <- field.as_text IF NOT empty
  RETURN profile, name
```

**Invariants**

- **A field is matched by name, not by position**, because the service is not obliged to
  return columns in the order they were asked for. The original optimizes for the common
  case where it does — it tries the next position first and falls back to a scan — which
  is an optimization, not a decision.
- **The count is read as a narrow integer and the date as a wide one**, matching the
  widths in [`profile_data_types.h`](profile_data_types.h.md). Reading a count as wide
  picks up whatever is beside it.
- A column the row does not carry leaves its field at zero, which is indistinguishable from
  a genuine zero. See [`profile_data_types.h`](profile_data_types.h.md) — the model has no
  absent state.

**Notes**

- **The name-column test is inverted.** The branch that assigns the player name fires when
  the field's name *differs* from the name column's, not when it matches — so the name is
  taken from whichever non-empty text field is seen last, and works only because the name
  column is the only text column in the request. A rebuild should compare for equality; as
  written, adding a second text column silently breaks every lookup.
- The comparison is also made against the *key* space's name for that field rather than the
  *stat* space's, which are different strings for the same concept. Combined with the
  inverted test, the two errors cancel. Both are worth knowing about before touching
  either.
