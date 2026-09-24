# src/xrEngine/XR_IOConsole.cpp

> The console: the registry of every named variable and command in the engine, the line that executes them, and the suggestion list.

**Needs** — [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`line_edit_control.h`](line_edit_control.h.md) · [`device.h`](device.h.md) · [`editor_base.h`](editor_base.h.md) · [`xr_input.h`](xr_input.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`EventAPI.h`](EventAPI.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`XR_IOConsole.h`](XR_IOConsole.h.md)
**Tier floor** — T2: a sorted name map, a fixed edit buffer, and drawing delegated to the overlay toolkit

## Purpose

Everything configurable in the engine is a *console command* — a named, typed, bounded
variable or an action. The set is not an implementation detail: the shipped game's
configuration files, its user settings, and every modification's scripts set these by name,
so **the names and their accepted argument syntax are frozen** (see the system
requirements, §6, criterion 3).

This file owns the registry, the execution of one line against it, the generation of
completion suggestions, and the drawing. The other four files in the cluster carry the
cursors, the key handling, the typed reads and the script surface.

## State

```text
RECORD Console
  commands        : map<text, Command>   # keyed by name, kept in name order. see note
  visible         : bool
  config_file     : text                 # the settings file this console writes
  edit_buffer     : text (1024 bytes)    # the line being typed

  history         : list<text>           # at most 64 entries, oldest dropped
  history_index   : int                  # -1 = not browsing; counts back from the newest
  last_command    : text                 # to suppress consecutive duplicates

  tips            : list<TipString>      # current suggestions; at most 220
  temp_tips       : list<text>           # a command's own argument suggestions, unfiltered
  tips_mode       : int                  # 0 none, 1 completing a command name,
                                         # 2 completing an argument of current_command
  current_command : text                 # the command whose arguments are being completed
  selected_tip    : int                  # -1 = none selected
  first_tip       : int                  # first visible suggestion
  tips_disabled   : bool                 # suppressed until the next edit
  previous_length : int                  # length of the edit buffer last time tips were built

RECORD TipString
  text              : text
  match_start, match_end : int           # the substring that matched, for highlighting

# invariant: commands is ordered by name, because completion walks it in order
#            and the "next command after this prefix" search is a lower-bound lookup
# invariant: selected_tip is -1 or a valid index into tips
# invariant: tips_mode says which of two substitutions Enter performs, so it must be
#            consistent with tips at all times
```

**Notes** — the registry is keyed by a bare name pointer compared as text, and ordered.
The ordering is not cosmetic: completion is implemented as a lower-bound search followed by
forward iteration, so it depends on the map being sorted by the same comparison the search
uses.

## The severity marks

**Contract** — a log line's first character may be a severity mark, and the console colours
the line by it. This is an in-band channel: the mark is part of the stored line, and readers
skip the mark plus one space when displaying it.

```text
ENUM ConsoleMark
  none  = ' '
  '~'   yellow          '!'  red     — error
  '@'   blue            — an echoed console command
  '#'   cyan            '$'  magenta      '%'  purple
  '^'   green           '&'  yellow       '*'  grey
  '-'   bright green    — success
  '+'   teal            '='  olive    — the console's own input echo
  '/'   light blue
```

Only three of these carry a documented meaning — error, echoed command, success. The rest
are a palette callers pick from ad hoc. A rebuild should replace the whole scheme with a
severity field on the log record and keep only the three meanings; it must still *parse*
the marks, because the shipped log lines and the dedicated-server console both use them.

## `Initialize`

**Contract** — reserves the history and tip buffers to their maxima so neither ever
reallocates while the console is open, attaches a handler for deferred console events, and
then calls the one function that registers the engine's entire command set. Registration is
a single call into [`xr_ioc_cmd.cpp`](xr_ioc_cmd.cpp.md) rather than a scatter of
constructors, so the set is enumerable and the order is fixed.

**Notes** — the console also declares itself the handler the crash reporter asks for the
user configuration file's name, so a crash report can name the settings in force.

## `AddCommand` / `RemoveCommand`

**Contract** — insert or erase by the command's own name. Adding a name that already exists
*replaces* it silently, which is how a game module overrides an engine command. Nothing
takes ownership: a command outlives its registration and unregisters itself.

## `ExecuteCommand`

**Contract** — runs one line. Splits it into the first word and the remainder, looks the
first word up, and dispatches. Unknown names and disabled commands are reported to the log
rather than failing. Echoes the line into the log and into the history when recording is
on; internal executions (from script, from a deferred event, from a configuration file) do
not record.

```text
FUNCTION execute_command(console, line, record)
  copy line INTO a scratch buffer
  reset the history cursor and the tip selection
  strip leading, trailing and repeated spaces
  IF the line is now empty
    RETURN

  IF record
    IF line differs from the last recorded command
      log it with the "console command" mark
      append it to the history
      remember it as the last command

  (first, rest) = split at the first space

  cmd = commands[first]
  IF cmd DOES NOT EXIST
    log "unknown command" WITH first
  ELSE IF NOT cmd.enabled
    log "command disabled"
  ELSE
    IF cmd.lowercases_its_arguments
      lowercase rest
    IF rest IS empty
      IF cmd.handles_empty_arguments
        cmd.execute(rest)
      ELSE
        log cmd.name AND cmd.current_status      # "show me the value"
    ELSE
      cmd.execute(rest)
      IF record
        cmd.remember_argument(rest)              # per-command recent-argument list

  IF record
    clear the edit buffer
```

**Notes** — the empty-argument branch is the load-bearing decision: typing a variable's name
with no argument *prints its value* rather than setting it to nothing. That single rule is
what makes the console usable as an inspector and is relied on by every troubleshooting
instruction ever written for this game. A command that genuinely wants to act on no
arguments opts out with a flag.

Suppressing a consecutive duplicate in the history means holding a key that re-executes does
not fill the history with one line. The per-command recent-argument list feeds argument
completion later.

Lowercasing arguments is per command because some take file paths (case-insensitively
matched, so lowering is safe and helps) and some take player names (where it is not).

## `Execute` / `ExecuteScript`

**Contract** — `Execute` is `ExecuteCommand` without recording: the entry point for script,
for deferred events and for configuration files. `ExecuteScript` is sugar that prefixes the
configuration-load command onto a filename, so a script can run a settings file by name.

## `OnEvent`

**Contract** — the deferred-execution path. A caller that cannot run a command *now* — a
script mid-frame, a thread that is not the main one — posts the line as an event; the
console executes it here, without recording, and releases the line's storage. This exists
because many commands reset the device or unload the level, which is illegal from inside
the code that asked.

## `OnFrame` — drawing

**Contract** — rebuilds the suggestion list every tenth frame, then, when visible, draws two
overlay windows: the console proper and the suggestion popup. Does nothing when hidden
except the periodic rebuild. Also re-captures input if the editor released it.

```text
FUNCTION on_frame(console)
  IF frame number is a multiple of 10
    update_tips(console)
  IF NOT visible
    RETURN
  IF the editor is not capturing input
    keep the text-input mode in sync
    IF input is not currently ours
      capture it

  in_game = a level is ready OR the main menu is up; never on a dedicated server

  # the console window: full width; half height when in game, full height otherwise
  draw a scrolling log view, one line per entry, coloured by its severity mark,
      clipped to the visible rows and auto-scrolled while the view is at the bottom
  draw the prompt, then the edit field, then the total line count

  IF the edit field reports Enter
    IF no suggestion is selected
      execute_command(edit_buffer, record = true)
    ELSE
      replace the edit buffer with the selected suggestion:
        mode 1 -> the suggestion plus a trailing space        # completing a command name
        mode 2 -> current_command, a space, then the suggestion   # completing an argument
      clear the selection
    re-focus the edit field

  IF in_game AND suggestions exist AND they are not suppressed
    draw the suggestion popup anchored under the edit field,
        each row highlighted over the matched substring,
        the selected row scrolled into view once
```

**Notes** — rebuilding suggestions every tenth frame rather than on every keystroke is a
cost decision: the rebuild scans the whole registry twice (see `add_internal_cmds`) and at
typing speed a tenth-of-a-frame granularity is imperceptible. The consequence is a visible
lag between the last character typed and the list updating, which the tip-suppression flag
exists partly to hide.

The console occupies half the screen in game and all of it otherwise, because with no level
loaded there is nothing behind it worth seeing.

Auto-scroll follows the bottom only while the view is already at the bottom, so reading back
through the log is not interrupted by new lines.

The suggestion popup is drawn only in game. On a dedicated server the native text console
draws instead (see [`Text_Console.cpp`](Text_Console.cpp.md)).

## `Show` / `Hide`

**Contract** — showing clears the edit buffer, resets both cursors, rebuilds the
suggestions, captures input and joins the frame sequence. Hiding reverses it and hands the
text-input mode back. A dedicated server refuses to hide the console — it is the only
interface there is.

**Notes** — joining and leaving the frame sequence rather than testing visibility inside a
permanently registered handler means a hidden console costs nothing per frame except the
periodic tip rebuild, which runs from elsewhere.

## `IR_OnKeyboardPress`

**Contract** — the console claims exactly two keys and forwards everything else to the
overlay toolkit, which owns the edit field. The quit action closes the *suggestion list*
first and the console second, so a player who opened a suggestion list by accident gets one
press to dismiss it rather than losing the line they were typing.

```text
FUNCTION on_key_press(console, key)
  action = the action this key is bound to
  IF action IS quit AND a suggestion is selected
    suppress the suggestions AND RETURN
  IF action IS quit OR action IS console
    hide()
    RETURN
  forward the key to the overlay toolkit
```

**Notes** — the console is bound by *action*, not by scancode, so it follows the player's
key bindings. Key release, key hold and text input are forwarded wholesale.

## `update_tips`

**Contract** — rebuilds the suggestion list from the current edit buffer. Decides between
two modes: completing a command *name*, or completing an *argument* of a command already
named. Resets the selection whenever the buffer's length changed, so typing does not leave a
stale row highlighted.

```text
FUNCTION update_tips(console)
  clear tips and temp_tips; current_command = none
  IF NOT visible
    RETURN
  IF the edit buffer is empty
    previous_length = 0; RETURN
  IF the buffer's length changed since last time
    reset the selection
  previous_length = the buffer's length

  (first, rest) = split the buffer at the first space

  # argument mode: a plausible command name followed by a space
  IF length(first) > 2 AND the buffer continues past it AND the next character is a space
    cmd = commands[first]
    IF cmd EXISTS
      mode = 0
      IF the buffer has two spaces there
        mode = 1                     # a second space asks for the *alternate* suggestion set
        skip one character of rest
      temp_tips = cmd.suggestions(mode)
      tips_mode = 2; current_command = first
      tips = filter temp_tips BY rest, recording where each matched
      IF tips is empty
        tips = ["(empty)"]
      RETURN

  # name mode
  tips = every command name matching the whole buffer
  tips_mode = 1
  IF tips is empty
    tips_mode = 0; reset the selection
```

**Notes** — three details are load-bearing. The three-character minimum on the command name
stops a lone "a " from being treated as a command. The *double space* selecting an alternate
suggestion set is how a command offers two lists — typically "values you may set" versus
"values you have set recently"; it is undiscoverable and a rebuild should give it a real
syntax. The `(empty)` placeholder row exists so that a command with no suggestions shows
*something*, distinguishing "this command takes no completable argument" from "the console
did not understand you".

## `add_internal_cmds`

**Contract** — collects every command name matching a fragment, in two passes: first those
that *begin* with it, then those that merely *contain* it. Each suggestion records the
matched range so the drawing can highlight it. Stops at the maximum suggestion count.
Duplicates between the passes are rejected.

```text
FUNCTION collect_matching(fragment, out suggestions) -> bool
  FOR EACH name IN commands              # pass 1: prefix matches, case-insensitive
    IF name begins with fragment AND name is not already present
      append (name, matched 0..length(fragment))
    IF suggestions is full
      RETURN
  FOR EACH name IN commands              # pass 2: substring matches, case-sensitive
    IF name contains fragment AND name is not already present
      append (name, matched at the position found)
    IF suggestions is full
      RETURN
```

**Notes** — the two passes exist so prefix matches sort first, which is what a person typing
a name expects; ordering the combined result would need a score. The passes differ in case
sensitivity — the prefix pass folds case, the substring pass does not. That is almost
certainly an oversight rather than a decision, and a rebuild should fold both.

## `find_next_cmd`

**Contract** — given a fragment, finds the first command at or after it in name order. Used
by tab completion, which walks the registry forward from wherever the typed text falls.
Understands one prefix: a line beginning with the remote-administration prefix has that
prefix held aside and re-attached to the result, so completion works inside a remotely
issued command.

**Notes** — the search appends a space to the fragment before the lower-bound lookup, which
makes an exact name match land on the *next* command rather than on itself. That is what
makes repeated tab presses cycle forward instead of sticking.

## `add_next_cmds`

**Contract** — walks forward from a fragment collecting consecutive command names until the
suggestion budget is full. Present but unused: the name-mode path calls
`add_internal_cmds` instead, and the call here is commented out in the source. Its loop is
bounded by twice the suggestion maximum as an explicit protection against the walk failing
to advance.

## `select_for_filter`

**Contract** — filters a command's own suggestion list by a fragment, recording where each
match occurred for highlighting. An empty fragment admits everything.
