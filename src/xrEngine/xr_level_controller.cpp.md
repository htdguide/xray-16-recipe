# src/xrEngine/xr_level_controller.cpp

> The binding layer: the frozen table of named actions, the frozen table of named keys, the three-slot map between them, and the console commands that edit it.

**Needs** — [`xr_level_controller.h`](xr_level_controller.h.md) · [`xr_input.h`](xr_input.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_ioc_cmd.h`](xr_ioc_cmd.h.md) · [`StringTable/StringTable.h`](StringTable/StringTable.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`xr_level_controller.h`](xr_level_controller.h.md)
**Tier floor** — T2: two parallel tables and a small map; the only hard constraint is that lookups happen per input event.

## Purpose

Nothing in the engine reads a key. Everything reads an **action** — `jump`, `wpn_fire`,
`ui_accept` — and this file is the only place that knows which physical key produces which.
It owns three frozen vocabularies and the mapping between two of them:

- the **action names**, addressed by shipped scripts, by the interface layer and by
  `user.ltx`;
- the **key names**, addressed by the same files (`kW`, `mouseL`, `gpA`);
- the **scancodes**, which are the platform's, and which the engine stores rather than
  characters, because a binding must survive a keyboard-layout change.

On top of that it adds two orthogonal filters — a *group* (single-player, multiplayer, or
both) and a *context* (none, interface, map, conversation) — which together let the same
physical key mean different things in different situations without the reader knowing it.

## State

```text
RECORD Action
  name        : text          # FROZEN. the identifier in user.ltx and in scripts
  id          : int           # index into the action table; see the ordering invariant
  group       : Group         # single-player only / multiplayer only / both
  context     : Context       # none / interface / map / conversation

RECORD Key
  name        : text          # FROZEN. the identifier in user.ltx, e.g. "kW", "mouseL"
  scancode    : int           # the platform's; stable across keyboard layouts
  display     : text          # what the player is shown; see Remap

RECORD Binding
  action      : Action
  keys        : list<optional<Key>>      # exactly 3 slots; see below

RECORD BindingText                        # cached, so the interface need not rebuild it
  keyboard_mouse : text                   # "Primary, Secondary", or "not bound"
  controller     : text

GLOBAL
  bindings         : list<Binding>        # one per action, indexed BY action id
  binding_text     : list<BindingText>    # parallel to bindings
  current_group    : Group                # which session kind is running
  console_bindings : map<scancode, text>  # a key that runs a console line instead
```

**Invariants**

- **The action table's order must equal the action identifier order.** The identifier *is*
  the index into both the action table and the binding table, and the tables are authored by
  hand in two places. A mismatch is checked at startup with a message that names the
  offending action and tells the maintainer exactly what they forgot. That check is the most
  useful line in the file; a rebuild that derives one table from the other deletes both the
  check and the failure mode.
- Exactly three key slots per action, and their meanings are fixed by position:
  **0 = primary keyboard or mouse, 1 = secondary keyboard or mouse, 2 = gamepad.** Every
  binding command, the settings file and the display text depend on that order.
- A key may appear in at most one action *within a conflicting group-and-context pair*; see
  the conflict rules.

## The two filters

Two predicates, each with a *matching* form (used when dispatching an event) and a
*conflict* form (used when deciding whether a new binding must evict an old one). They are
not each other's negation, and the difference is the subtle part of the file.

```text
GROUPS      single-player | multiplayer | both

groups_match(a, b)       = a == b OR either is "both"
groups_conflict(a, b)    = NOT (one is single-player AND the other is multiplayer)

CONTEXTS    none | interface | map | conversation

contexts_match(a, b)     = a == b            # "none" matches only "none"
contexts_conflict(a, b)  = a != b, OR both are "none"
```

**Notes** — read the two conflict rules together with the two match rules.

**Group:** an action marked "both" *matches* anything, so it fires in either kind of
session. But only a single-player/multiplayer *pair* is non-conflicting, so the same key may
be bound to a single-player action and a multiplayer action simultaneously — the session in
force decides which fires. Anything involving "both" conflicts, because it would fire in
both sessions.

**Context:** matching is exact and "none" does not match a context. So an action with no
context never fires while the interface, the map or a conversation is up, and a
context-specific action never fires during play. That is what lets `kQ` be "lean left" in
the world and "previous tab" in the interface with no arbitration anywhere else in the
engine.

**The conflict rule for contexts is written wrong, or at least surprisingly:**
`contexts_conflict` reports a conflict when the two contexts *differ*, and also when both
are "none". Read literally, binding a key in the interface context evicts a world binding —
which is the opposite of what the match rule makes possible. In practice the eviction is
guarded by group conflict *and* context conflict together (both must hold), so the common
cases survive; but two actions in different contexts and compatible groups do evict each
other. **This looks like an inverted condition rather than a decision**, and a rebuild
should define eviction as "the same key, and the two actions could both fire in the same
situation", which is `groups_match AND contexts_match`.

## `GetBindedAction` — the dispatch lookup

**Contract** — given a scancode and the current context, returns the action it fires, or
"not bound". Linear over every action; runs once per input event.

```text
FUNCTION binded_action(scancode, context) -> Action id
  FOR EACH binding IN bindings
    IF NOT groups_match(binding.action.group, current_group)   THEN CONTINUE
    IF NOT contexts_match(binding.action.context, context)     THEN CONTINUE
    FOR EACH slot IN binding.keys
      IF slot EXISTS AND slot.scancode == scancode
        RETURN binding.action.id
  RETURN not bound
```

**Notes** — linear, over roughly two hundred actions, per key event. That is fine at human
input rates and would not be fine at any other rate; a rebuild keying a map by
(scancode, context) is strictly better and changes no behaviour, provided it preserves
**first match wins in action-table order**, which is what makes the hand-authored table
order a tie-breaker.

## `IsBinded`, `GetActionDik`, `ForAllActionKeys`

**Contract** — the reverse direction, asked by anything that wants to *show* a binding.
`IsBinded` tests whether a specific action is bound to a specific key in a specific context.
`GetActionDik` returns one slot's scancode, or with no slot named, the first bound slot in
order — so "the key for this action" means primary, else secondary, else gamepad.
`ForAllActionKeys` visits every bound slot, optionally stopping early.

## Name and identifier lookup

**Contract** — four linear searches over the two frozen tables: name to action, identifier to
name, key name to key, scancode to key. Each reports a miss to the log in non-shipping
builds unless silenced, and returns nothing. Missing names are *not* fatal: a settings file
naming an action this build does not have must be ignored, not rejected, because
configuration files outlive builds.

## `RemapKeys` — the display names

**Contract** — asks the platform what each key is *called on the keyboard currently in use*
and stores it as the display name; falls back to the internal name. Then rebuilds every
binding's cached display text. Re-run whenever the platform reports the keyboard layout
changed, through a watcher registered at high priority.

**Notes** — the separation of the three names per key is the decision: the **scancode** is
what is stored and compared, the **internal name** is what appears in configuration files,
and the **display name** is what the player is shown. Only the third changes with the
layout. Storing display names in the settings file, or matching on characters, would break
every binding the moment a player switched layouts — which is precisely the bug this shape
avoids.

The watcher is only installed the first time a binding command runs, which means a build
that never binds anything never watches. Harmless, and a rebuild can install it
unconditionally.

## `TranslateBinding` — the cached display text

**Contract** — builds the two strings the interface shows for one action: the keyboard and
mouse slots joined by a comma when both are bound, and the gamepad slot alone. An unbound
side is the localized "not bound" string rather than empty.

**Notes** — cached rather than computed on demand because the interface asks for it per
widget per frame, and it changes only on a rebind or a layout change. The localized
placeholder means the interface never has to test for emptiness.

## `GetActionBinding`

**Contract** — returns the display text appropriate to **the input device the player last
used**, asked of the input layer. This is why an on-screen prompt says "F" on a keyboard and
shows the gamepad's face button a moment after the player picks up a controller, with no
setting involved.

## The binding commands

All of these are console commands, which is how bindings persist: the settings file is a
list of `bind` lines.

**`bind` / `bind_sec` / `bind_gpad`** — bind an action to a key in slot 0, 1 or 2. Ignores an
unknown action or key silently. After binding, walks every *other* action and clears any
slot holding the same key where the two conflict by group **and** by context. The first call
of any of them lazily performs the initial key remap and installs the layout watcher.

**`bind` also writes the whole binding table when the settings file is saved**, and slot 0's
command writes a `default_controls` line first. That is the decision that makes a settings
file self-repairing: loading it re-establishes the defaults before applying the overrides,
so an action added by a newer build is bound rather than dead.

**`unbind` / `unbind_sec` / `unbind_gpad`** — clear one slot of one action.

**`unbindall`** — clear every slot of every action and drop every console binding.

**`default_controls`** — clear everything, then apply the built-in default table, then
execute the shipped `default_controls` configuration file. Two layers, and the order is
load-bearing: the built-in table binds only what the engine itself must have bound to be
operable — movement, fire, use, the interface's accept and back, the editor key — and the
shipped file binds the rest. The built-in half is the floor that guarantees a player can
always get back to a menu; the file is the part a modification may replace.

Note the built-in defaults **only fill empty slots**, testing each before writing. Since the
command clears everything first, that test can only matter if a slot were already filled —
which it cannot be. Dead caution.

**`list_actions`** — print every action name. **`bind_list`** — print every action with its
three display names, showing an absent slot explicitly. The two together are the
discoverability surface for the binding vocabulary.

**`bind_console` / `unbind_console`** — bind a **console line** to a key, keyed by scancode
in a separate map consulted before the action map. The argument order is inverted relative
to `bind`: the command line comes first and the key name last, because the command may
contain spaces and the key never does, so the last token is unambiguous. Saved as its own
lines.

## The built-in default bindings

**Contract** — the minimum map. Read it as a statement of what the engine considers
non-negotiable rather than as a preference.

```text
                    primary        secondary      gamepad
look / move         —              —              right stick / left stick
fire / aim          mouse 1        —              right trigger / left trigger
jump                space          —              A
crouch toggle       —              —              B
reload              R              —              X
use                 F              —              Y
torch               L              —              right stick click
scores              tab            —              left stick click
inventory           I              —              right shoulder
active jobs         P              —              left shoulder
map                 M              —              —
contacts            H              —              —
accept / cancel     return         —              start / back
quick use 1-4       F1-F4          —              d-pad up/left/right/down
editor              F10            —              —

interface           W A S D + arrows, return/escape, Q/E for tabs, 1-0 for buttons,
                    and the gamepad's face buttons and d-pad
map screen          arrows to pan, Z/C or keypad +/- to zoom, X to reset,
                    R to centre on the player, V for the legend, B for the filters
conversation        X to switch to trade, Q/E or page up/down to scroll the log
```

**Notes** — the interface and map contexts bind *both* the letter cluster and the arrow
cluster to the same actions, in the two keyboard slots, which is how the menus are usable
without the player learning which the game expects. The gamepad column is populated for
every action a controller-only player needs and left empty for the rest, which is the
honest statement that this game is not fully playable on a controller.

## `initialize_bindings`

**Contract** — links each binding to its action by index, after asserting the two tables
agree. In a debug build also reports any two key names sharing a scancode, which is the
other way the hand-authored tables go wrong.

## `CCC_RegisterInput` / `CCC_DeregisterInput`

**Contract** — install the twelve binding commands and initialize the tables; on the way out,
release the layout watcher if it was installed. Called from the console's initialization.
