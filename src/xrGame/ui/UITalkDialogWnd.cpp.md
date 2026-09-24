# src/xrGame/ui/UITalkDialogWnd.cpp

> The dialogue screen's widgets: a scrolling log of what has been said, a list of what can be said
> next, two portraits, and a button that is either "trade" or "upgrade".

**Needs** — [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md) · [`UITalkWnd.h`](UITalkWnd.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`../game_news.h`](../game_news.h.md) · [`../Level.h`](../Level.h.md) · [`../Actor.h`](../Actor.h.md) · [`../alife_registry_wrappers.h`](../alife_registry_wrappers.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Buttons/UIBtnHint.h`](../../xrUICore/Buttons/UIBtnHint.h.md)
**Used by** — [`UITalkDialogWnd.h`](UITalkDialogWnd.h.md)
**Tier floor** — T3: list construction and text layout; nothing device-facing

## Purpose

Everything the player sees while talking to a character. It holds no conversation state at all: the
question list is refilled from outside, the answer log is appended to from outside, and the only
thing it reports back is *which* question was clicked — by notification, with the identifier left in
a field for the conversation half to read.

The screen is built to serve two shipped layout vocabularies at once. The older game's dialogue
screen is two framed lines with titles; the newer games' is two background pictures with separate
name labels. Nearly every element here is optional, and the file's shape is mostly the consequence
of that.

## State

```text
RECORD DialogueScreen extends Window
  document        : LayoutDocument        # kept alive: list items are built from it on demand
  conversation    : ConversationHalf      # the CUITalkWnd above; used only to request a stop

  answers         : ScrollContainer       # the log; appended to, auto-scrolled to the end
  questions       : ScrollContainer       # refilled whenever the available phrases change
  frame_top       : optional<Widget>      # hosts the answers; carries the other's name in SOC
  frame_bottom    : optional<Widget>      # hosts the questions; carries our name in SOC

  our_portrait    : optional<Widget>      # holds a character-info panel
  others_portrait : optional<Widget>

  trade_button    : Button                # reads "trade" or "upgrade"
  exit_button     : optional<Button>
  button_slots    : [both_left, both_right, centre]   # three authored positions

  clicked_question : text                 # the identifier of the last question clicked
  mechanic_mode    : bool                 # the trade button means upgrade
  name_font, name_colour, our_replics_colour
```

Invariants:

- The layout document is retained for the screen's whole life, because every question and answer row
  is constructed from a named template in it at the moment it is added.
- `clicked_question` is written immediately before the click notification is sent and read
  immediately after it is received; it is a one-slot channel, not a record.
- The three button positions are captured at build time from where the layout put the two buttons;
  the centre slot is computed as the midpoint of the two and is used when only one button shows.

## `InitTalkDialogWnd`

**Contract** — Builds the screen, tolerating either layout vocabulary. Falls back to the full canvas
when the layout has no root window element. Allocates; runs once per session.

```text
FUNCTION InitTalkDialogWnd()
  document <- load "talk.xml"
  IF document has no "main" window THEN self.rect <- the whole virtual canvas

  optional statics: "top_background", "bottom_background"

  # which side of the screen is whose portrait differs between games
  our_tag, others_tag <- "right_character_icon", "left_character_icon"
  IF running the oldest game's data THEN swap them
  our_portrait    <- optional static our_tag;    attach a character-info panel filling it
  others_portrait <- optional static others_tag; attach a character-info panel filling it

  # newer layouts name the two frames; the older one has two indexed frame lines
  frame_bottom <- optional static "frame_bottom"
  frame_top    <- optional static "frame_top"
  IF frame_top    IS none THEN frame_top    <- optional frame line "frame_line_window" index 0
  IF frame_bottom IS none THEN frame_bottom <- optional frame line "frame_line_window" index 1

  answers   <- scroll container "answers_list"   inside frame_top    (or the screen)
  questions <- scroll container "questions_list" inside frame_bottom (or the screen)

  trade_button <- button "button";     button_slots[0] <- its position
  exit_button  <- optional button "button_exit"
  IF exit_button EXISTS THEN
    button_slots[1] <- exit_button.position
    button_slots[2] <- (midpoint of the two x positions, trade_button's y)
  ELSE
    button_slots[1] <- button_slots[2] <- button_slots[0]

  name_font, name_colour       <- "font" index 0
  our_replics_colour           <- "font" index 1

  bind: any widget named "question_item" sending LIST_ITEM_CLICKED -> question clicked
  bind: trade_button BUTTON_CLICKED -> trade clicked
  bind: exit_button  BUTTON_CLICKED -> stop the conversation
```

**Notes** — The **creation order of the two frames is reversed between the two vocabularies**, and
the source says why: draw order is list order, so the frame created second is drawn on top. In the
newer layouts the questions frame must be under the main dialogue frame, in the older one above it.
Building them in the order the file does reproduces both.

The portrait swap exists because the oldest game puts the player on the left and the newer ones on
the right. Nothing else in the file distinguishes the games.

The question binding is by **widget name**, not by widget, because question rows are created and
destroyed constantly — binding by name means the binding survives every refill. The name
`question_item` is therefore load-bearing in two places: it is the layout template's element name
*and* the name assigned to every row.

The frozen element names are `main`, `top_background`, `bottom_background`,
`left_character_icon`, `right_character_icon`, `frame_top`, `frame_bottom`, `frame_line_window`,
`answers_list`, `questions_list`, `button`, `button_exit`, `font`, and the item templates
`question_item`, `actor_answer_item`, `other_answer_item`.

## `Show` / `Hide`

**Contract** — Showing announces the information portion `ui_talk_show` to the player character,
resets the whole subtree, and **locks navigation to the question list** so directional navigation
cannot wander onto the trade button or out of the screen. Hiding unlocks it (only if this screen
locked it), announces `ui_talk_hide`, and discards any pending button tooltip.

**Notes** — Announcing an information portion on open and close is how scripts observe that the
player is talking; the two identifiers are part of the shipped script contract.

## `AddQuestion`

**Contract** — Appends one question row built from the `question_item` template, carrying the
phrase identifier that will be reported when it is clicked. Numbers the first ten rows and gives
each a digit accelerator. A row marked as a conversation *finalizer* — one that ends the exchange —
additionally answers the UI back action and the use action, so "goodbye" is always on the same keys.

```text
FUNCTION AddQuestion(text, phrase_id, index, is_finalizer)
  row <- new QuestionItem from document["question_item"]
  row.init(phrase_id, text)
  n <- index + 1                                  # the list is zero-based, the labels are not
  IF n <= 10 THEN
    row.number_label <- decimal(0 IF n == 10 ELSE n) + "."
    row.accelerator <- the digit key for n
  IF is_finalizer THEN
    row.accelerator <- UI back action
    row.accelerator <- use action
  row.name <- "question_item"                     # so the name-keyed binding matches
  questions.insert(row)
  register row so its notifications reach this screen
```

**Notes** — The tenth row is labelled `0` because that is the key that picks it, the same convention
as the skin picker. Accelerators are added at distinct priorities so a finalizer's three bindings do
not displace one another.

## `AddAnswer`

**Contract** — Appends one utterance to the log from one of two templates — the player's or the
other speaker's — scrolls the log to the end, and **also files the line as a news entry in the
player's personal log**, tagged as talk, stamped with the current game time and the speaker's
portrait. So every conversation is readable afterwards in the PDA.

The logged text is prefixed with an inline colour escape fixing the body colour; the caption carries
the speaker's name separately.

**Notes** — The news side effect is the reason this screen reaches into the player character's alife
registry at all. A rebuild that splits "show it" from "record it" must keep both happening on the
same call, because nothing else records the conversation.

## `AddIconedAnswer`

**Contract** — Two forms, both appending a picture-bearing log row from a caller-named template, and
both also filing a news entry. The first takes a registered icon name; the second takes a texture
plus an explicit sub-rectangle, which is how an inventory item's grid cell is shown. Used when the
conversation transfers an item, so the log shows what changed hands.

## `SetOurName` / `SetOthersName`

**Contract** — Set the speaker names, **only in the older layout vocabulary**, where they are the
title text of the two framed lines. In the newer vocabulary the frames are plain pictures with no
title and both calls do nothing — the names are shown by the character-info panels instead.

## `SetOsoznanieMode`

**Contract** — Switches the screen into a stripped presentation used for a specific scripted
sequence: both portraits, the answer log, the top frame and the trade button are all hidden,
leaving only the question list. Also — regardless of the mode argument — sets the trade button's
caption and tooltip from `mechanic_mode`: `ui_st_upgrade` / `ui_st_upgrade_hint` for a mechanic,
`ui_st_trade` / `ui_st_trade_hint` otherwise, each tooltip applied only if the localization actually
has it.

**Notes** — The two jobs in one function are not related; the caption assignment is here because
this is the last call made before the screen is shown. A rebuild may separate them.

## `UpdateButtonsLayout`

**Contract** — Decides which of the two buttons are visible and where they sit. Trade is visible
when trading is enabled; exit is visible unless the conversation forbids breaking off. With both
visible they take their authored positions; with exactly one, that one moves to the **centre slot**,
so a lone button is not left off to one side.

```text
FUNCTION UpdateButtonsLayout(disable_break, trade_enabled)
  trade_button.shown <- trade_enabled
  IF exit_button IS none THEN RETURN
  exit_button.shown <- NOT disable_break
  IF both shown        THEN trade -> slot[0], exit -> slot[1]
  ELSE IF exit shown   THEN exit  -> slot[2]
  ELSE IF trade shown  THEN trade -> slot[2]
```

**Notes** — Called every frame from the conversation half, because whether trade is enabled can
change mid-conversation.

## Question navigation

**Contract** — Directional navigation through the question list, built on the toolkit's geometric
focus search rather than on list indices, because question rows have varying heights and the list
scrolls.

```text
FUNCTION FocusOnNextQuestion(forward, may_wrap)
  focused <- the currently focused widget
  IF focused is not inside a question row THEN FocusOnFirstQuestion(); RETURN
  candidate <- nearest focusable below (or above) the focused widget's centre
  IF candidate is inside a question row THEN focus it
  ELSE IF may_wrap THEN focus the first (or last) question
```

**Notes** — Wrapping is allowed on a discrete key press and refused on a held key, so holding a
direction walks to the end of the list and stops rather than cycling forever. The caller passes that
distinction in; see [`CUITalkWnd::OnKeyboardAction`](UITalkWnd.cpp.md).

The search returns a primary and a fallback candidate; the fallback is used when the primary is
missing, which is how navigation continues across a gap in the list's geometry.

## `OnKeyboardAction`

**Contract** — While the cursor is over the question list, the UI accept action activates the
focused question row — turning a controller press into the same click path a pointer takes.
Everything else falls through.

## `CUIQuestionItem`

**Contract** — One question row: a full-width text button and an optional number label, built from a
named template. Its height is the greater of the template's minimum and the wrapped text's extent,
so a long question grows its row. Clicking the text sends a **list-item-clicked** notification
carrying the row, which the screen's name-keyed binding catches.

```text
FUNCTION Init(phrase_id, text)
  self.phrase_id <- phrase_id
  text_button.text <- text
  text_button.fit_height_to_text()
  self.height <- max(template_min_height, text_button.y + text_button.height)
```

## `CUIAnswerItem`

**Contract** — One logged utterance: a name caption and a body, both from a named template. Height
is the greater of the template minimum and the wrapped body, plus an authored bottom footer that
separates consecutive utterances. Owns itself — it is marked for automatic deletion, because the log
is cleared wholesale.

## `CUIAnswerItemIconed`

**Contract** — An answer row with a picture to its left. Two initialisation forms:

- name plus text: the two are **joined into one string** with an explicit newline escape and an
  inline colour escape between them, and passed to the base as the body with an empty caption — so
  the icon row's name and text share one wrapped text block rather than two widgets;
- text plus an explicit texture sub-rectangle: used for inventory item pictures, where the
  rectangle is the item's cell in the icon atlas.

Both stretch the picture into its authored rectangle.

**Notes** — Folding the name into the body is what makes the text wrap *around* the icon correctly:
one text block with a known width can be laid out beside the picture, two stacked widgets could not
without a second layout pass.
