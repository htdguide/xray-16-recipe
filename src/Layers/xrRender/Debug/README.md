# `src/Layers/xrRender/Debug/` — frame markers for GPU capture

One idea, one file. Every significant stage of the frame — the g-buffer fill, each shadow
cascade, the light accumulation, each post-process — opens a named marker on the command
list it records into, and closes it when that stage ends. A GPU capture tool reads those
markers and shows the frame as a labelled tree, which is the difference between a capture
an engineer can navigate and four thousand anonymous draw calls.

## Where it sits

A leaf of chapter 18 with one dependency upward, the command backend, and one outward: the
[Profiler and GPU debugging seam](../../../../SYSTEM-REQUIREMENTS.md#seam-profiler-and-gpu-debugging),
which that seam's own entry marks *given, optional*. Nothing depends on this directory —
markers are written into the render path and read by nobody in the process.

**It is optional in a rebuild.** The whole mechanism compiles to nothing in a shipping
build, and that is the decision worth keeping: the markers sit in inner loops, they cost a
driver call each, and the engine refuses to make a player pay for them. A rebuild may omit
the directory entirely and lose only tooling.

Two facts survive if you do keep it. Markers must **nest and never overlap**, because an
unbalanced marker corrupts the capture's tree for the rest of the frame — the original gets
this structurally by tying the close to a scope's end, and a rebuild with explicit begin and
end calls has to guarantee it another way. And a marker belongs to **one specific command
list**: the renderer records several in parallel, so a marker opened on one and closed on
another is a silent cross-thread bug.

| File | Role |
|---|---|
| [`dxPixEventWrapper.h`](dxPixEventWrapper.h.md) | Opening and closing a named region on a command list, and hiding the fact that the two graphics APIs expose the facility differently |
