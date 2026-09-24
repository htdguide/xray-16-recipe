# src/xrGame/Message_Filter.cpp

> A tap on the message stream: decodes just enough of each message to identify it, hands matching ones to a registered observer, and logs a readable trace of everything it saw.

**Needs** — [`Message_Filter.h`](Message_Filter.h.md) · [`NET_Queue.h`](NET_Queue.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md)
**Used by** — [`Message_Filter.h`](Message_Filter.h.md)
**Tier floor** — T1: it reads a wire format positionally and must restore the read cursor exactly

## Purpose

Two problems, one mechanism. Demo playback needs to notice when a particular message goes past
without consuming it — the first spawn message, so playback can be restarted from there. And
debugging a multiplayer session needs a readable log of which messages arrived in what order.
Both are "look at the stream without changing it", so both live here.

The observer is keyed by the pair *(message kind, sub-kind)*, because the interesting
distinctions are inside a message: every entity event is the same message kind and it is the
event sub-kind that says whether this is a destruction or an ownership transfer.

## State

```text
RECORD MessageKey                    # the observer key, and the partially decoded message
  msg_type     : int (16-bit)        # the message kind
  msg_subtype  : int (32-bit)        # the event or game-event kind within it; 0 when absent
  dest_obj_id  : int (16-bit)        # the addressed entity, for events
  msg_receive_time : int (32-bit)

  ORDER BY (msg_type, msg_subtype)   # the other two fields take no part in comparison

RECORD MessageFilter
  filters          : map<MessageKey, observer>
  log_file         : writer or none
  last_line        : text            # for run-length collapsing of the log
  repeat_count     : int
```

**Invariants** — only the kind and sub-kind participate in the key's ordering. The addressed
entity and the receive time ride along on the same record purely so the decode step can
produce them in one pass; if they took part in comparison, no lookup would ever match.

Registration asserts that the key is not already present, and removal asserts that it is. The
filter is not a multi-subscriber bus — one observer per key, and a missing one is a
programming error, not a condition.

## `check_new_data`

**Contract** — examines one message without consuming it. Decodes its kind and sub-kind, logs
it, and if an observer is registered for that pair, calls it with the message positioned for
reading. Restores the read cursor to exactly where it was on entry, so the caller's own
handling is unaffected. A packed batch of events is unpacked and each member examined in turn.

**Invariants** — the cursor is saved on entry and restored on exit unconditionally. This is the
whole contract: the filter sits *before* the real dispatch in
[`Level_network_Demo.cpp`](Level_network_Demo.cpp.md), and a filter that left the cursor moved
would corrupt every message it inspected.

```text
FUNCTION check_new_data(message)
  saved = message.read cursor
  key = decode(message)

  IF key.msg_type == EVENT_PACK THEN
    # Look inside the batch, at each member.
    WHILE bytes remain
      length = read one byte
      member = read that many bytes
      key = decode(member)
      REQUIRE key.msg_type != EVENT_PACK        # batches do not nest
      log(member, key)
      IF an observer is registered for key THEN call it with (kind, sub-kind, member)
    END WHILE
  ELSE
    log(message, key)
    IF an observer is registered for key THEN call it with (kind, sub-kind, message)
  END IF

  message.read cursor = saved
```

**Notes** — the observer is handed the message with its cursor positioned *after* the header
the decode step consumed, which is what lets an observer read the message's body. But the
cursor is then rewound for the caller, so an observer that reads must not assume its reads
persist. That asymmetry is subtle and is the file's sharpest hazard.

## `decode` (`msg_type_subtype_t::import`)

**Contract** — reads a message's identifying header, positionally, filling the key. Knows the
layout of exactly two message kinds and treats every other kind as having no sub-kind.

```text
FUNCTION decode(message) -> MessageKey
  msg_type = read the message identifier
  msg_subtype = 0
  SELECT msg_type
    CASE EVENT
      msg_receive_time = read 32 bits
      msg_subtype      = read 16 bits      # the event kind
      dest_obj_id      = read 16 bits      # the entity the event is addressed to
    CASE GAMEMESSAGE
      msg_subtype      = read 32 bits      # the game-event kind
  END SELECT
```

**Invariants** — this is a *second, partial* parser for a format whose real parser lives
elsewhere. The field order here must match the writer's exactly, and nothing enforces that.
A rebuild should decode once, into a record, and let both the dispatch and the filter read
that record — which deletes this function and its hazard.

Note that the receive time is only populated for entity events; for every other kind it is
left at whatever the record was constructed with, which is why the log's timestamps are zero
for most message kinds.

## `filter` · `remove_filter`

**Contract** — register and unregister one observer against a (kind, sub-kind) pair. Each
asserts the absence or presence of the key. No return value: failure is a crash.

## `dbg_print_msg`

**Contract** — formats a one-line description of a message, prints it, and appends it to the
log file when one is open. Consecutive identical lines are collapsed: the repeat is counted
and only flushed as a count when the line finally changes.

**Invariants** — the run-length collapsing writes the *previous* line's repeat count before
writing the new line, so the count trails its line by one entry. A log that ends mid-run loses
its final count.

**Notes** — formatting is per message kind, with named cases for the events worth reading at a
glance (entity destruction, ownership take and reject; player killed, round started, artefact
taken) and a numeric fallback for everything else. Two cases read *further* into the message
to print an operand, which is only safe because the caller rewinds afterwards.

Encountering a packed batch here is a hard failure rather than a fallback, because
`check_new_data` is supposed to have unpacked it first; reaching this case means the unpacking
was skipped.

The whole logging half is compiled only into builds with multiplayer logging enabled, and a
rebuild may drop it. The observer half is not optional — demo restart depends on it.

## `dbg_set_message_log_file`

**Contract** — opens the named file for writing. Logs and continues on failure rather than
failing: losing the trace must not stop the session.
