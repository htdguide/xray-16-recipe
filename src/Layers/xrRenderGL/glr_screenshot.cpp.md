# src/Layers/xrRenderGL/glr_screenshot.cpp

> Reads the finished frame back and writes it to a file.

**Needs** — [`glHW.h`](glHW.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it reads device memory back into a host buffer with an explicit pixel layout.

## Purpose

The engine's screenshot facility has several modes; this backend implements one of them. The page is short because the file is, and the honest content is which modes are missing and why that matters.

## `screenshot(mode, name)`

**Contract** — in the normal mode, reads the whole display surface back as three-channel eight-bit pixels and writes it as a compressed image at maximum quality under a generated name. Blocks: a read-back synchronizes the pipeline. Logs on write failure rather than faulting. The save-thumbnail mode is unimplemented; every other mode aborts in a checked build.

```text
FUNCTION screenshot(mode, name) -> ()
  SELECT mode
    normal ->
        filename := "ss_" + user name + "_" + timestamp
                    + "_(" + current level's name, or the menu's, + ").jpg"
        writer := open_for_write("$screenshots$", filename); FAIL IF none
        pixels := read back the display surface as 3-channel 8-bit
        IF NOT encode_and_write(pixels, quality = maximum, flip = true) THEN
            log "failed to make a screenshot"
        close(writer)
    save thumbnail -> (unimplemented)
    otherwise -> FAIL in a checked build
```

**Invariants** — the image is written **vertically flipped**. The read-back yields rows bottom-up because this API's window origin is bottom-left; the file format expects top-down. This is the same origin disagreement that mirrors every scissor and clear rectangle in [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md), surfacing one last time at the exit.

**Notes** — the missing mode is the one the save system uses: a small thumbnail stored inside a save game. Its absence means saves made on this backend carry no preview image. A rebuild should implement it — the work is a scaled read-back into a fixed-size buffer, and the target size is declared in the file (128 pixels square) even though nothing uses it.

A second pair of dimensions is declared for a multiplayer screenshot-upload mode that the dead matchmaking seam used to consume; nothing reads them.
