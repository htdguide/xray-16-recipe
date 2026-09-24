# src/xrGame/ui/UIMMShniaga.cpp

> The main menu's rotating drum: a vertical band of captions that spins behind a fixed lens, with cogs that turn at exactly the rate the band travels, a logarithmic ease-out, and a mechanical sound cued off the same motion.

**Needs** — [`UIMMShniaga.h`](UIMMShniaga.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`MMSound.h`](MMSound.h.md) · [`MainMenu.h`](../MainMenu.h.md) · [`xrUICore/ScrollView/UIScrollView.h`](../../xrUICore/ScrollView/UIScrollView.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`saved_game_wrapper.h`](../saved_game_wrapper.h.md) · [`Actor.h`](../Actor.h.md) · [`Level.h`](../Level.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIMMShniaga.h`](UIMMShniaga.h.md)
**Tier floor** — T3.

## Purpose

The first thing the player sees. It is not a list of buttons: it is a **drum**. The captions
ride a band that slides vertically; a lens sits fixed over one position; selecting an entry
moves the *band*, not the highlight, until the chosen caption is under the lens. Two cogs at
the sides turn as the band passes them, and a mechanical sound is cued from the same motion.

Reproducing the menu means reproducing that, and the four numbers that make it feel right are
the substance of this page.

## State

```text
RECORD MenuDrum EXTENDS Window, DeviceResetListener
  band          : Picture          # the moving strip; the cogs are its children
  lens          : Lens             # fixed; drawn in a post-process pass, not in the tree
  cogs          : two optional Pictures
  gratings      : two optional Pictures
  entries_view  : ScrollView       # holds the captions of the current page
  pages         : three caption lists — main, new game, network game
  page          : which of the three is shown
  selected      : index into the current page

  # motion
  start_time    : int (ms)
  run_time      : int (ms)         # computed per move
  origin        : real             # band position at the start of the move
  destination   : real             # where it must end
  lens_home_x   : real
  band_offset   : real             # authored fine adjustment of the rest position
  flags         : { sound_cued, motion_stopped }
```

**Invariants**

- **The band moves; the lens does not.** Every position on the page is derived from the
  band's offset, including the cogs' rotation and the click cue.
- A new selection can only be started once the previous motion has **stopped**. Queuing moves
  would desynchronise the sound and let the band overshoot.
- The three caption lists are built once and owned by the drum; showing a page clears the
  view and re-adds that page's captions.

## Which main menu

**Contract** — the main page's captions are read from one of five authored lists, chosen by
what the game is doing:

```text
FUNCTION which_main_list()
  IF no level is loaded
    IF there is no last save, or it is not loadable THEN "menu_main"
                                                    ELSE "menu_main_last_save"
  ELSE IF single player
    IF the player is dead THEN "menu_main_single_dead" ELSE "menu_main_single"
  ELSE "menu_main_mm"
```

**Notes** — the menu's *shape* is game state, not a set of disabled entries. A dead player
does not see a greyed-out "resume"; the entry is not on the drum. The save's validity is
actually checked, not merely its existence, so a corrupt last save removes the continue entry
rather than offering a crash.

The network page is loaded as **not required** — a build or a data set without multiplayer
simply has no such list, and the entry that would lead there does nothing.

## The motion

**Contract** — selecting an entry computes a move and the update drives it:

```text
FUNCTION begin_move(entry)
  play the spinning sound
  origin      := band.y
  destination := entry.y - lens.y + band_offset
  run_time    := (ln(1 + |origin - destination|) / ln(drum_height)) * 300 ms,
                 floored at 100 ms
  clear sound_cued and motion_stopped

FUNCTION position_at(t)            # t is milliseconds since the move began
  IF t >= run_time THEN fraction := 1
  ELSE fraction := ln(1 + 10 * t / run_time) / ln(11)
  RETURN origin + sign(destination - origin) * fraction * |destination - origin|
```

**Notes** — two logarithms, doing two different jobs.

*The easing curve* is logarithmic and normalised so that it reaches exactly 1 at the end: the
band leaves fast and settles slowly, which is how a heavy wheel with friction behaves. A
linear or a symmetric ease would read as a slide, not a spin.

*The duration* is also logarithmic in the distance, which is the less obvious of the two: a
move of ten times the distance does not take ten times as long. The drum spins *faster* for a
longer move, so stepping one entry and jumping to the far end feel like the same gesture. The
divisor is the natural log of the drum's own height, which makes the duration scale-free — the
same formula gives the same feel whatever size the drum is authored at. The 300-millisecond
coefficient and the 100-millisecond floor are the tuning; no derivation is recoverable and a
rebuild should keep both.

## The cogs

**Contract** — each cog's rotation is derived from the band's position as though the cog were
rolling along it:

```text
circumference := 2 * pi * (cog.height / 2)
angle         := 2 * pi * (band.y modulo circumference) / circumference
left cog.heading  := -angle
right cog.heading := +angle
```

**Notes** — this is correct rolling-contact geometry, not a decorative spin: the angle is
band travel divided by cog radius. The two cogs counter-rotate because they sit on opposite
sides of the band. Getting this wrong is immediately visible — a cog that turns at the wrong
rate reads as broken machinery, which is exactly the impression the menu is built to avoid.

The modulo keeps the angle bounded over a long band; without it the accumulated value would
lose precision.

Both cogs and both gratings are optional; a game whose menu art has no machinery simply omits
them.

## The sound

**Contract** — three cues, driven by the same motion state: the spin starts with the move, a
single mechanical click fires **once, within the first tenth of the run time**, and the spin
stops when the band reaches its destination. The background music is pumped every frame.

**Notes** — the click is cued near the *beginning* of the move despite being raised by a step
named for finalisation, and the condition that raises it is the same comparison as the motion
test with a tenth of the duration. It reads as an inverted test: a detent click belongs at the
end of a spin, not at its start. It is what the shipped menu does, so a rebuild reproducing
the original keeps it; one fixing it should move the cue to the last tenth.

Stopping also **snaps the band to the exact destination**, because the easing curve is
evaluated against a clock that can overshoot by a frame.

## The lens

**Contract** — the lens is **not drawn by the widget tree**. It registers itself with the main
menu's post-process pass and hides itself from ordinary drawing; the pass draws it over the
finished frame. Hiding it is done by **moving it off the canvas** — one unit past the right
edge — rather than by clearing its shown flag, because the post-process pass does not consult
that flag.

**Notes** — the lens is a distortion over what is behind it, so it must be composited after
the scene, which no widget in chapter 15 can express. Registering with a separate pass is the
escape hatch. The off-canvas trick is the consequence: a rebuild whose post-process draw
respects visibility deletes it. The parking coordinate, 1025, is one past the canvas width.

## Input

**Contract** —

- A pointer press **inside the lens** activates the selected entry — clicking the *lens*, not
  the caption. A caption receiving pointer focus selects it, which slides the drum under the
  pointer.
- The accept action activates; the back action returns to the main page from either
  sub-page.
- Up and down step the selection with **wrap-around at both ends**, and are handled on the
  *hold* channel while the press channel consumes them silently.
- A step is refused while the drum is still moving.
- The controller's directional axis does the same as up and down.

**Notes** — the press/hold split is a workaround: the input layer delivers both a press and a
hold for one physical keystroke within a single frame, so acting on the press would step
twice. Consuming the press and acting on the hold gives one step per keystroke and gives
key-repeat for free. The source records this reason. It also keeps the toolkit's geometric
navigation focus from acting on the same key, which would step a third time.

Refusing input while moving is the same rule as refusing to queue moves, seen from the key
side.

## Activating an entry

**Contract** — two caption names are handled **by the drum itself** — the one that opens the
new-game page and the one that returns from it — and every other caption is dispatched as an
ordinary button-clicked notification to the drum's message target.

**Notes** — those two names are frozen strings matched against the caption's authored element
name. They are special-cased because paging is the drum's own business and everything else is
the menu screen's. A rebuild that routes all three through notifications must keep the two
names, because the shipped documents use them.
