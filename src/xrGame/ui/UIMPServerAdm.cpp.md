# src/xrGame/ui/UIMPServerAdm.cpp

> The administrator's server page: a one-level menu of four alternative panels that turn every
> control on them into a single remote-administration console command.

**Needs** — [`UIMPServerAdm.h`](UIMPServerAdm.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIDialogWnd.h`](UIDialogWnd.h.md) · [`xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [`xrUICore/Buttons/UICheckButton.h`](../../xrUICore/Buttons/UICheckButton.h.md) · [`xrUICore/EditBox/UIEditBox.h`](../../xrUICore/EditBox/UIEditBox.h.md) · [`xrUICore/SpinBox/UISpinNum.h`](../../xrUICore/SpinBox/UISpinNum.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`UIMPServerAdm.h`](UIMPServerAdm.h.md)
**Tier floor** — T3: visibility toggling and command-string assembly.

## Purpose

The widest of the three admin pages, and the one that shows the whole architecture of remote
administration in miniature: **the screen holds no server state**. Every control is a write —
press it and a command goes out — and nothing on the page ever reads a value back except the
check boxes, which read the *local* console variables at construction time.

It is one file because the panels share one back button and one visibility rule; splitting
them would multiply that rule by four.

## State

```text
RECORD ServerAdminPage EXTENDS Window
  back_btn        : Button          # shown only while a sub-panel is open
  main_panel      : Window          # restart, fast restart, three sub-panel entries, vote stop
  weather_panel   : Window          # four named weathers plus a change-rate stepper
  game_type_panel : Window          # four game modes
  limits_panel    : Window          # four value limits, five spectator modes, three timers,
                                    # five behaviour flags
```

**Invariants**

- Exactly one of the four panels is visible. The three sub-panels start hidden and the main
  panel starts visible; entering a sub-panel hides the main panel and shows the back button,
  and the back button restores the main panel and hides all three sub-panels unconditionally.
  A rebuild may hold the current panel as one value instead of four visibility flags, which
  makes the invariant structural.
- Every control inside a panel reports to the *page*, not to its containing panel. A panel is
  purely a visibility group; it handles nothing. That is why each control's message target is
  redirected at construction.
- The back button's visibility doubles as the "a sub-panel is open" predicate the owning
  screen queries. A rebuild that hides chrome differently must keep an explicit predicate.

## `Init`

**Contract** — configure this window, the four panels and every control from the `server_adm`
section tree of the already-open layout document, then have each of the ten check boxes pull
its current value from the console variable it is bound to. Element paths are frozen by the
shipped document.

**Notes** — only the check boxes are preloaded. The text fields start empty by design: they
are *commands with an argument*, not editors of a current value, so showing the server's
present setting in them would be misleading — the page cannot read the server's settings, only
the local client's.

## Command emission

**Contract** — each control maps to exactly one remote-administration command. A button emits
it on click; a check box emits it with its new state as the argument; a text field emits it
with its contents as the argument and then clears itself, and emits nothing when empty. The
four commands that change what the match *is* — full restart, fast restart, and each of the
four game-mode switches — additionally close the whole admin screen, because the page they
were pressed on will be meaningless a moment later.

```text
FUNCTION on_click(control)
  IF control IS back_btn                THEN leave_sub_panel()
  ELSE IF control IS a sub_panel_entry  THEN enter(that sub panel)
  ELSE IF control IS a plain_command    THEN remote("<its command>")
  ELSE IF control IS a mode_switch      THEN remote("<its command>"); close_screen()
  ELSE IF control IS a check_box        THEN remote("<its command> " + (checked ? 1 : 0))
  ELSE IF control IS a set_button       THEN
    text <- its paired field.text
    IF text IS empty THEN RETURN          # an empty field is not a command
    remote("<its command> " + text)
    its paired field.clear()
```

**Notes** — three details do not follow the pattern and are load-bearing.

- **The four weather buttons set a time of day, not a weather.** Clear, cloudy, rain and night
  are nine, thirteen, sixteen and one o'clock. The weather system is a keyframed day cycle, so
  naming a weather *is* naming an hour, and the four hours are the ones at which the shipped
  environment definition has those conditions. The mapping is data-dependent: a rebuild that
  changes the shipped environment file breaks these four buttons and nothing warns it.
- **The vote-enable check box sends 255, not 1.** The variable is a bit mask of which vote
  kinds are permitted, and the check box means "all of them". A rebuild that treats it as a
  boolean silently narrows the feature.
- **The weather change-rate stepper commits through its own button**, like the text fields and
  unlike the check boxes, so a rate is only sent when confirmed.

## Panel navigation

**Contract** — entering a sub-panel hides the main panel, shows the back button and shows that
sub-panel. Leaving shows the main panel, hides the back button and hides all three sub-panels.
Neither transition touches the page's own visibility; the owning screen does that.
