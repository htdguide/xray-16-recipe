# src/xrUICore/MessageBox/UIMessageBox.cpp

> Builds a modal dialog's child set from a style named in data, wires standard accelerators onto whichever buttons exist, and translates each button's click into a style-specific answer message sent twice — once naming the button, once naming the dialog.

**Needs** — [`UIMessageBox.h`](UIMessageBox.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`XML/xrUIXmlParser.h`](../XML/xrUIXmlParser.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`EditBox/UIEditBox.h`](../EditBox/UIEditBox.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIMessageBox.h`](UIMessageBox.h.md)
**Tier floor** — T3.

## Purpose

One file, three decisions: what a style means in terms of children; what the accelerator
convention is; and what a click turns into.

## State

```text
RECORD MessageBox EXTENDS Static
  style          : ENUM of the ten styles
  button_yes_ok  : optional<Button>    # the affirmative button, whatever it is called
  button_no      : optional<Button>
  button_cancel  : optional<Button>
  button_copy    : optional<Button>
  picture        : optional<Static>
  message_text   : optional<Static>
  caption_host, caption_pass, caption_user_pass : optional<Static>
  edit_host, edit_pass, edit_user_pass, edit_url : optional<Edit>
  return_value   : text                # scratch for the assembled host string
```

**Invariants**

- Every child is optional and every accessor tolerates its absence, because the style decides
  which exist. Nothing asserts that a style produced the children it should have.
- The children are *not* auto-delete: they are released explicitly by `Clear`, which also runs
  before a rebuild and from the destructor. Rebuilding a dialog into a different style is
  therefore legal.

## `init_message_box`

**Contract** — loads the shared dialog document, navigates to the named template, and fails
(reporting false, not aborting) if it is absent. Builds the optional picture and message-text
children if their sub-elements exist, then initializes the dialog's own static from the
template element itself, reads the mandatory `type` attribute, maps it to a style, and builds
the style's child set. Finally applies the accelerator convention.

```text
FUNCTION init_message_box(template) -> bool
  document <- load the shared message-box document
  IF template not present RETURN false

  IF element(template, "picture")       EXISTS THEN build picture
  IF element(template, "message_text")  EXISTS THEN build message text
  initialize self from element(template)

  style <- map of the "type" attribute      # required; absent is fatal
  MATCH style
    ok             : button_ok
    info           : no children at all
    yes_no,
    quit_windows,
    quit_game      : button_yes, button_no
    yes_no_cancel  : button_yes, button_no, button_cancel
    yes_no_copy    : button_yes, button_no, button_copy, and an optional url field
    direct_ip      : host caption+field, password caption+field, button_yes, button_no
    password       : user-password caption+field, password caption+field,
                     button_yes, button_no
    ra_login       : login caption+field, password caption+field,
                     then FALL THROUGH into yes_no's buttons

  apply_accelerators()
  RETURN true
```

**Notes** — the element names under a template are themselves frozen: `picture`,
`message_text`, `button_ok`, `button_yes`, `button_no`, `button_cancel`, `button_copy`,
`cap_host`, `edit_host`, `cap_password`, `edit_password`, `cap_user_password`,
`edit_user_password`, `cap_login`, `edit_login`, `edit_url`.

The login style chains into the yes/no style's button construction rather than repeating it,
and additionally links its two fields into a tab chain and gives the first one the keyboard
immediately — the only style that opens focused.

The `info` style has no buttons at all and is dismissed by whoever owns it.

## `apply_accelerators`

**Contract** — the accelerator convention, applied to whichever buttons exist.

```text
affirmative : slot 2 <- the "enter" action, slot 3 <- the UI accept action
negative    : slot 3 <- the UI back action
              slot 2 <- the "quit" action, but only when there is no cancel button
cancel      : slot 2 <- the "quit" action, slot 3 <- the first UI action
copy        : slot 2 <- the first UI action
```

**Notes** — slots 2 and 3 are used and slots 0 and 1 are left free for the data to fill; the
`ok` style pre-loads slot 1 with the quit action before running the element's own
initialization, precisely so the data can override it. That slot discipline is the whole
mechanism by which a shipped layout can rebind a dialog's escape key without losing the
engine's defaults.

Giving the *quit* action to the negative button only when no cancel exists is what makes
escape mean "no" in a two-button dialog and "cancel" in a three-button one.

## `on_yes_ok` and `send_message`

**Contract** — a click on one of the dialog's buttons is translated into a style-specific
answer and announced **twice**: once with the button as the sender and once with the dialog as
the sender. The affirmative button's answer depends on the style — an acknowledgement for the
`ok` and `info` styles, a yes for the question styles, and a dedicated quit-to-desktop or
quit-to-menu answer for the two quit styles, which are announced only once, from the dialog.

```text
FUNCTION on_affirmative()
  MATCH style
    ok, info       : announce OK_CLICKED        from the button, then from self
    quit_windows   : announce QUIT_WIN_CLICKED  from self only
    quit_game      : announce QUIT_GAME_CLICKED from self only
    otherwise      : announce YES_CLICKED       from the button, then from self
```

**Notes** — the double announcement exists because the callback mixin binds by *sender*: a
screen may have bound a handler to the button (which it can name from the XML) or to the
dialog (which it holds). Sending both means either binding works. A rebuild with a single
dispatch point should announce once and let the screen decide, but must then re-check every
shipped screen's bindings.

## `get_host`

**Contract** — reads the host field and rewrites `address:port` into the engine's own
`address/port=NNNN` connect-string form, leaving a bare address untouched. The result lives in
the dialog's scratch string and is valid until the next call.

**Notes** — this is the only content transformation any dialog performs, and it exists because
the connect command's argument syntax is not the syntax a player types.

## `set_password_mode` / `set_user_password_mode`

**Contract** — show or hide one password row — its caption and its field together — so that one
built dialog can serve both the "server password" and the "server plus user password" cases
without rebuilding.

## `set_text` / `get_text` / the field accessors

**Contract** — the body text is set through the localization table; reading it returns the
translated string. Every field accessor returns nothing when its style did not create the
field.
