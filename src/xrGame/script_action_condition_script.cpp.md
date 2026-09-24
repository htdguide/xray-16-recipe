# src/xrGame/script_action_condition_script.cpp

> Exports the action-end condition to the script virtual machine under the name scripts spell it: `cond`.

**Needs** — [`script_action_condition.h`](script_action_condition.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Scripts construct a condition inline whenever they issue an action, so this surface is one of
the most frequently typed in the shipped script set. Its names are frozen by conformance
criterion 10.

## State

`Stateless.`

## `script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up:

- **the condition type**, under the short name `cond`, with three constructors: empty, a flag
  set, and a flag set plus a life time;
- **its flag names**, nested inside the type so scripts address them as members. The exported
  names describe the *event* rather than the part — movement is exported as "movement ended",
  the head-turn as "look ended", and so on — because a script writes them as the things it is
  waiting for.

**Notes**

- Seven of the eight flags are exported. The particle flag is registered nowhere and so
  cannot be named from script; see
  [`script_action_condition.h`](script_action_condition.h.md).
- The flags are exported as plain integers rather than as a distinct type, so scripts combine
  them with ordinary arithmetic or bitwise operations. A rebuild that exports a stricter type
  will break shipped scripts that add flags together.
