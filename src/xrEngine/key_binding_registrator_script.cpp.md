# src/xrEngine/key_binding_registrator_script.cpp

> Exports the action vocabulary, the key-context vocabulary and every scancode to the script layer, by name.

**Needs** — [`xr_level_controller.h`](xr_level_controller.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`xr_level_controller.h`](xr_level_controller.h.md)
**Tier floor** — T3: a table of name-to-number pairs handed to an interpreter.

## Purpose

One registration function whose entire body is a list of names and their numeric values.
It exists as its own file because the list is long enough to dominate anything it shares a
file with, and because the compiler is slow on the registration form used.

Everything here is **frozen**. Shipped scripts index these tables by name — `key_bindings.kUSE`,
`DIK_keys.DIK_F1` — and a rebuild that renames one breaks a script it did not write. The
acceptance criterion in the system requirements that every shipped script must load
unmodified fixes this file exactly.

## The script surface

```text
TABLE key_bindings_context.context
    undefined · ui · pda · talk               # the four key contexts

TABLE key_bindings.commands
    one entry per action, named "k" + the action's enumerator spelling:
    kLOOK_AROUND, kFWD, kWPN_FIRE, kUSE, kUI_ACCEPT, kPDA_MAP_ZOOM_IN, ...

TABLE DIK_keys.dik_keys
    one entry per physical input:
    DIK_A .. DIK_Z, DIK_0 .. DIK_9, DIK_F1 .. DIK_F24, the keypad,
    the navigation and editing cluster, the modifiers, the international keys,
    the media and browser keys, the brightness and keyboard-illumination keys,
    MOUSE_1 .. MOUSE_5,
    GAMEPAD_A/B/X/Y, GAMEPAD_BACK/GUIDE/START,
    GAMEPAD_LEFTSTICK/RIGHTSTICK, GAMEPAD_LEFTSHOULDER/RIGHTSHOULDER,
    GAMEPAD_DPAD_UP/DOWN/LEFT/RIGHT,
    GAMEPAD_DPAD_MISC1, GAMEPAD_DPAD_PADDLE1..4, GAMEPAD_DPAD_TOUCHPAD

FUNCTION dik_to_bind(scancode)            -> action id
FUNCTION dik_to_bind(scancode, context)   -> action id
FUNCTION bind_to_dik(action)              -> scancode of the first bound slot
FUNCTION bind_to_dik(action, slot)        -> scancode of that slot (0/1/2)
```

**Notes** — the names are **not** a mechanical transform of anything. Three seams show:

- The action names carry the `k` prefix of the engine's internal enumerator, not the
  lower-case spelling the settings file uses. A script says `kUSE`; `user.ltx` says `use`.
  Two spellings of one vocabulary, both frozen, and a rebuild must ship both.
- The key names carry a `DIK_` prefix naming a *different* input library than the one in
  use: they are inherited from the engine's original input layer, and several are now
  aliases onto scancodes whose meaning shifted. The gamepad and paddle names keep an
  inherited `DPAD_` in their spelling even for buttons that are not on the d-pad
  (`GAMEPAD_DPAD_PADDLE1`, `GAMEPAD_DPAD_TOUCHPAD`) — a copy-paste that is now frozen.
- One name ships with a stray closing parenthesis inside it
  (`DIK_KBDILLUMTOGGLE)`). It is exported that way and must stay that way; anything that
  used it used the typo.

The four free functions are the only *behaviour* here, and each is a one-line forward into
the binding layer. Both overloaded pairs exist because the script layer resolves overloads
by argument count at call time — a script may ask for a binding with or without naming a
context or a slot.

The engine's own action count is not exported, so script cannot enumerate actions; it can
only name the ones it was written against. That is a real limitation for a modification
that wants to present a bindings screen, and the reason the interface layer builds its own
list instead.
