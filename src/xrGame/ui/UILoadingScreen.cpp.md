# src/xrGame/ui/UILoadingScreen.cpp

> The screen shown while a level loads: a background, a level logo, a progress bar and a rotating tip — driven from the loader's thread, so every field it touches is taken under a lock.

**Needs** — [`UILoadingScreen.h`](UILoadingScreen.h.md) · [`UILoadingScreenHardcoded.h`](UILoadingScreenHardcoded.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`xrEngine/ILoadingScreen.h`](../../xrEngine/ILoadingScreen.h.md) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`UILoadingScreen.h`](UILoadingScreen.h.md) · [`UILoadingScreenHardcoded.h`](UILoadingScreenHardcoded.h.md)
**Tier floor** — T2: it is mutated from a loading thread and drawn from the render thread, and the mutual exclusion is explicit.

## Purpose

The engine owns the loading sequence and knows nothing about widgets; it drives an abstract
loading-screen interface. This file is the game layer's implementation of that interface — the
one place in this chapter where the UI is a *service the engine consumes* rather than a screen
the player opens.

Two decisions carry it: **the layout has a built-in fallback**, so a loading screen exists
even with no UI data at all, and **every entry point is serialised**, because the loader calls
from its own thread.

## State

```text
RECORD LoadingScreen EXTENDS Window, LoadingScreenInterface
  lock              : Mutex
  always_show_stage : bool          # authored override of the user's setting
  progress          : ProgressBar
  progress_percent  : optional<Label>
  level_logo        : Picture
  stage             : optional<Label>    # what is being loaded right now
  header, tip_number, tip : optional<Label>
```

**Invariants**

- **Every public method takes the lock**, including draw. The loader thread writes the stage
  text and the progress while the render thread draws; without this the text layer would be
  read mid-rewrite.
- Progress is set **forcibly**, bypassing the bar's inertia. The bar normally eases toward
  its target over several frames, and loading frames are not regular — easing would make the
  bar lag arbitrarily behind. Recorded in the source as temporary until loading joins the
  ordinary frame loop.
- Hiding the screen **destroys the level logo's material**. It is a full-screen texture that
  is never needed again, and the loading screen is precisely the moment the memory is needed
  for the level.

## The built-in layout

**Contract** — the layout is loaded from data if it exists. If it does not, a **compiled-in
document** is used instead, chosen by mounted game and by whether the display is widescreen,
together with a compiled-in *texture description* that must be registered first because the
document references icons by name.

**Notes** — this is the strongest form of the three-games pattern in the chapter: rather than
requiring each game's data to supply a loading screen, the engine carries six of them. The
reason is bootstrapping — the loading screen is needed before anything else is known to work,
and a UI-data problem would otherwise present as a black screen during load.

Expressing the fallback as **an XML document in the binary** rather than as constructed
widgets is itself the decision worth keeping. The document goes through the same reader as
shipped data, so the fallback cannot drift from the real path, and a mod that ships a document
overrides it wholesale. A rebuild keeps an embedded document, not embedded layout code.

## Per-language elements

**Contract** — the background and the level logo are each looked up twice: first under a name
suffixed with the current language code, then under the plain name. The suffixed element wins
when it exists.

**Notes** — some games ship a loading screen with text baked into the image. Per-language
*elements*, rather than per-language *textures*, means the localized variant can also differ
in geometry. The lookup is by element name, which makes the language code part of the frozen
layout vocabulary.

## Element order

**Contract** — an authored flag decides whether the progress bar is created **before** or
**after** the background.

**Notes** — chapter 15 states that z-order is list position and nothing else. This flag is
that rule exposed to data: some games' loading art has a recess the bar sits inside, and some
have the bar floating over the art. One boolean, two stacking orders, no depth value anywhere.

## Updating

**Contract** — progress is `completed / total` scaled to the bar's authored range; the
percentage label, when present, shows the bar's own position rounded to a whole number.

**Notes** — the label reads back from the bar rather than recomputing, so the two can never
disagree even if the bar clamps.

## The stage title

**Contract** — the stage text is written only when the player has enabled loading-stage
display **or** the layout forces it on, and only if the element exists.

**Notes** — a per-game default expressed in data, overriding a user setting in the *permissive*
direction only. The one game whose loading screen is designed around the stage line forces it
on; elsewhere it is opt-in.

## `NullLoadingScreen`

**Contract** — an implementation of the same interface that does nothing and reports itself
never shown.

**Notes** — used by the dedicated server, which loads levels with no graphics device at all.
The engine's loading sequence is unchanged; only the implementation differs. That is what
makes the interface worth having.
