# src/xrCore/Debug/dxerr.h

> Declares the graphics/audio error-code decoder: a numeric result in, a symbolic name or a sentence out.

**Needs** — [`dxerr.cpp`](dxerr.cpp.md) · [`../xrCore.h`](../xrCore.h.md)
**Used by** — [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md) · [`DXGetErrorString.inl`](DXGetErrorString.inl.md) · [`DXTrace.inl`](DXTrace.inl.md) · [`dxerr.cpp`](dxerr.cpp.md)
**Tier floor** — T1: the codes it decodes are a platform's own packed 32-bit result values, and it decodes them by numeric identity.

## Purpose

Declares the surface implemented in [`dxerr.cpp`](dxerr.cpp.md). This is a vendor diagnostic library adopted wholesale, and the only reason the engine carries it is that the platform's own error-to-text service does not know the graphics, audio or media codes — so a failed device creation would otherwise log a bare hexadecimal number.

## Exported units

- **`DXGetErrorString(code)`** — the code's *symbolic name*, as text. Returns a pointer to static storage; never fails, returning the literal `Unknown` for an unrecognized code.
- **`DXGetErrorDescription(code, buffer, capacity)`** — a human-readable sentence, copied into the caller's buffer. Copies rather than returning static storage, deliberately — see the twin.
- **`DXTrace(file, line, code, message, show_dialog)`** — write a formatted diagnostic to the debug output, optionally raise a modal dialog offering to break into a debugger, and **return the code unchanged** so the call can wrap an expression.
- **`DXTRACE_MSG` / `DXTRACE_ERR` / `DXTRACE_ERR_MSGBOX`** — call-site wrappers that capture the file and line, and that compile to nothing but the code itself in a release build.

**Notes** — every entry point exists in two character-width flavours selected by a build switch. That is entirely incidental: it is one function whose string type is a build decision, and a rebuild has one function.

Returning the code unchanged from the trace is the decision worth keeping: it means a call site wraps its result rather than restructuring around the diagnostic, and it is why the release-build wrappers can collapse to the bare expression without changing the surrounding code's shape.
