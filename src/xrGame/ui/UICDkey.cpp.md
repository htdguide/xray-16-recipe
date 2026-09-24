# src/xrGame/ui/UICDkey.cpp

> The product-key field: how a key is masked while it is typed, and where a key and a player
> name live when the game is not running.

**Needs** — [`UICDkey.h`](UICDkey.h.md) · [`../../xrUICore/Lines/UILines.h`](../../xrUICore/Lines/UILines.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UICDkey.h`](UICDkey.h.md)
**Tier floor** — T1: per-machine settings store and a hand-rolled text draw

## Purpose

Two things the game must remember **outside its own data**, because they identify the machine
and the person rather than the save: the multiplayer product key and the player's name. Both
live in a per-machine settings store. This file is the only place that store is read or
written for these two values, and it is also where the key's masked presentation lives.

The service the key authenticates against is dead
([Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)),
so the field still works and still validates locally, and nothing it produces reaches anyone.

## State

```text
RECORD KeyField
  backup      : text    # the settings protocol's undo value
  view_access : bool    # may the real characters be shown rather than masked
```

Invariant: `view_access` becomes true when the field is empty — you can watch yourself type a
key you are entering for the first time — and false again once the entered key validates. A
key already accepted is never shown again.

## The stored format versus the typed format

**Contract** — The key is **stored with hyphens and edited without them**. Every read from the
store strips them; every write re-inserts them; the accessor that returns the field's text for
display re-inserts them. The two conversions are a matched pair and the file would be
incoherent without both.

**Invariants** — The grouping is four characters per group, and the caret arithmetic in `Draw`
hardcodes the same grouping by adding one hyphen's width past character 4, 8 and 12. The
format is frozen by the store's existing contents and by whatever once validated it.

## `Draw` — masking

**Contract** — The field draws its own text rather than letting the text layer do it, because
what is drawn is not what is stored.

```text
FUNCTION draw()
  IF the edit buffer is empty THEN view_access = true

  draw the background as a plain picture (not as a text widget)
  place the text at the vertical centre of the field, at the alignment indent

  # The mask is a run of 'x' the same LENGTH as the buffer, so the field
  # still shows how much has been typed.
  masked      = "x" repeated buffer length, capped at 63
  masked_head = "x" repeated (characters before the caret), capped at 63

  IF the field has input focus
    draw hyphenate(view_access ? buffer : masked)

    # Place the caret by MEASURING the text before it, then adding one
    # hyphen's width for each group boundary already crossed.
    x = field left + width_of(view_access ? buffer_head : masked_head)
    IF caret index > 3  THEN x = x + hyphen width
    IF caret index > 7  THEN x = x + hyphen width
    IF caret index > 11 THEN x = x + hyphen width
    draw an underscore at x
  ELSE
    draw hyphenate(masked)                  # unfocused: always masked

  flush the font
```

**Invariants** — Measuring the mask rather than the real text is what keeps the caret in the
right place while masked: `x` and a real character are different widths in a proportional
font, so the caret would drift if the mask were drawn but the real text measured.

The 63-character cap is the buffer's size, not a key-length rule.

An unfocused field is masked **regardless of `view_access`**, so a key is never left visible
on screen.

## `paste_from_clipboard`

**Contract** — Bound to both the usual paste gestures. Reads the clipboard, **removes every
character that is not alphanumeric**, truncates to 16 characters, and replaces the field's
contents.

**Invariants** — Stripping rather than rejecting is the decision: a key copied from a web page
or an e-mail arrives with hyphens, spaces and newlines, and demanding the player clean it up
by hand is the behaviour this replaces. 16 is the un-hyphenated key length.

## The settings protocol

**Contract** — Four operations, as chapter 15 describes, against the per-machine store instead
of a console variable:

- **read current** — pull the key from the store and put its un-hyphenated form in the field;
- **back up** — remember the field's text;
- **commit** — re-hyphenate the field's text, write it to the store, and **clear `view_access`
  if the key now validates**;
- **undo** — restore the backup;
- **changed?** — compare the backup with the field.

`Show` re-reads the current value on every appearance, so the field never shows a stale key.
Losing focus commits — which is unusual for a settings control and means an accidental click
away saves the partial key.

## `CUIMPPlayerName::OnFocusLost`

**Contract** — On losing focus, fire the edit-committed notification and write the name to the
per-machine store. No settings-protocol participation, no undo.

## The four store operations

**Contract** — Read and write the key and the name.

Both key operations truncate at **64 characters**, on read and on write.

`GetPlayerName_FromRegistry` is the one with a platform split, and the split is real rather
than incidental:

```text
FUNCTION get_player_name()
  on the platform with a per-user settings store
    read the stored name; empty if absent
  on POSIX-family platforms
    # There is no game-specific store, so the player's identity is taken
    # from the operating system's own account record: the display name if
    # there is one (truncated at the first comma, the field being a
    # comma-separated record), otherwise the login name.
    read the current user's account record
    name = display name up to the first comma, or the login name

  IF empty THEN warn and RETURN
  truncate to the account service's maximum nickname length
  warn IF still empty
  pass the name through the name sanitiser, which is what actually
    enforces the character set
```

**Invariants** — The maximum length is the dead account service's nickname limit, kept so that
an existing stored name is not silently rejected. The sanitiser is a separate module and is
where the real character rules live; this function only length-limits.

The name defaults to the operating-system account on platforms with no store, which is a
genuine design decision: the multiplayer name is a *person's* identity, so falling back to the
machine's idea of who is logged in is the least surprising default.

`WritePlayerName_ToRegistry` truncates to the same maximum and stores.
