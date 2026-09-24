# src/xrGame/script_animation_action_script.cpp

> Exports the animation part to the script virtual machine under the name scripts spell it: `anim`.

**Needs** — [`script_animation_action.h`](script_animation_action.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Scripts construct an animation part inline every time they issue an action with a clip or a
posture in it, so this is one of the most frequently written surfaces in the shipped scripts.
Its names are frozen by conformance criterion 10.

## State

`Stateless.`

## `script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up:

- **the type**, under the short name `anim`, with five constructors: empty, a clip name, a
  clip name plus the movement-controller flag, a mental state, and a monster animation kind
  plus its variant index.
- **the mental states**, nested under a `type` name: free, danger, panic. Three only.
- **the monster animation kinds**, nested under a `monster` name: ten roles — stand idle,
  capture-prepare, sit idle, lie idle, eat, sleep, rest, attack, look around, turn.
- **two setters** exposed under short names: naming a clip, and naming a posture. Both are
  the setters described in
  [`script_animation_action_inline.h`](script_animation_action_inline.h.md), with their
  side effects on the completed flag intact — so calling the posture setter on a part that
  was waiting for a clip marks it finished.
- **the completed read**, inherited from the shared part interface.

**Notes**

- The variant index has no exported enumeration: a script asking for an attack animation
  passes a bare integer and must know from the model's animation bank how many variants
  exist. Out-of-range values are not validated here. This is the least discoverable part of
  the monster animation surface.
- The movement-controller flag is reachable only through the two-argument constructor; there
  is no setter for it. A script that builds the part any other way cannot turn it on.
