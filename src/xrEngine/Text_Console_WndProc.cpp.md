# src/xrEngine/Text_Console_WndProc.cpp

> The two platform message handlers for the dedicated server's console windows.

**Needs** — [`Text_Console.h`](Text_Console.h.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: a platform message loop callback

## Purpose

A native window is driven by messages, and both console windows need only two of them
answered. This is a separate file purely because it is the one place platform message
handling appears in the console; everything in it is compiled only on the platform that has
these windows.

## State

`Stateless.` Both handlers reach the single process-wide console instance.

## The two handlers

**Contract** — the outer window forwards its repaint message to the console and lets the
platform handle everything else. The inner log window does the same and additionally claims
the *erase-background* message as handled without doing anything, which suppresses the
platform's own background fill.

```text
FUNCTION console_window_message(message)
  IF message IS REPAINT
    console.on_paint()
    RETURN handled
  RETURN default handling

FUNCTION log_window_message(message)
  IF message IS ERASE_BACKGROUND
    RETURN handled                # claim it; do nothing. see note
  IF message IS REPAINT
    console.on_paint()
    RETURN handled
  RETURN default handling
```

**Notes** — suppressing the background erase is the load-bearing line. The platform would
otherwise fill the window with the class brush immediately before every repaint, and since
the repaint blits a complete image anyway, the fill is visible as a flash. Claiming the
message without acting is the standard way to say "I paint every pixel myself".

Both handlers reach the console through a downward type cast on the process-wide console
pointer, which is safe only because a dedicated server installs exactly this subclass. A
rebuild passes the instance through the window's own user data instead.
