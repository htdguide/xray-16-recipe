# src/xrCore/FS.h

> Declares the reader and writer surface over the chunked binary container.

**Needs** — [`FS.cpp`](FS.cpp.md) · [`FS_impl.h`](FS_impl.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_color.h`](_color.h.md) · [`xrstring.h`](xrstring.h.md)
**Used by** — [`TextureDescrManager.cpp`](../Layers/xrRender/TextureDescrManager.cpp.md) · [`dxThunderboltDescRender.cpp`](../Layers/xrRender/dxThunderboltDescRender.cpp.md) · [`dxUIRender.cpp`](../Layers/xrRender/dxUIRender.cpp.md) · [`dx11Texture.cpp`](../Layers/xrRenderDX11/dx11Texture.cpp.md) · [`xr_ini_ex.cpp`](../utils/mp_balancer/xr_ini_ex.cpp.md) · [`xr_ini_ex.h`](../utils/mp_balancer/xr_ini_ex.h.md) · [`mp_config_sections.cpp`](../utils/mp_configs_verifyer/mp_config_sections.cpp.md) · [`xrCompress.cpp`](../utils/xrCompress/xrCompress.cpp.md) · [`xrCompress.h`](../utils/xrCompress/xrCompress.h.md) · [`xrCompressDifference.cpp`](../utils/xrCompress/xrCompressDifference.cpp.md) · [`game_graph_inline.h`](../xrAICore/Navigation/game_graph_inline.h.md) · [`game_level_cross_table_inline.h`](../xrAICore/Navigation/game_level_cross_table_inline.h.md) · [`graph_abstract.h`](../xrAICore/Navigation/graph_abstract.h.md) · [`graph_abstract_inline.h`](../xrAICore/Navigation/graph_abstract_inline.h.md) · _and 47 more_
**Tier floor** — T1: the reader's public surface includes "give me a pointer to the current position and the byte count remaining", which callers use to cast a struct over mapped bytes.

## Purpose

Declares the surface implemented in [`FS.cpp`](FS.cpp.md): the writer, the reader, and the reader-over-a-writable-mapping. Two constants that belong to the *format* rather than to any one function live here — the chunk-identifier bit that marks a compressed payload (the top bit, `0x80000000`) and the identifier of the chunk that carries an archive's configuration header (666).

The quantization rules, the string conventions and the chunk framing are all contracted in [`FS.cpp`](FS.cpp.md), even though several of them are written as inline code in this file; they are one idea and are described in one place.

## Exported units

- **`IWriter`** — append-only-with-seekback byte sink, plus the chunk stack, the scalar and vector writes, the quantized writes, and formatted text output. Abstract: the flavours below fill it in.
- **`CMemoryWriter`** — writer over a doubling heap buffer; can hand out its bytes or spill itself to a path.
- **`IReaderBase`** — the read vocabulary shared by every reader: scalars, vectors, colours, quantized reals, packed directions, chunk lookup, and read-a-whole-chunk-into-this-buffer.
- **`IReader`** — the concrete cursor over a contiguous range, adding position/length/pointer access, the two string conventions, chunk opening and chunk iteration.
- **`CVirtualFileRW`** — a reader over a file mapped *writable and shared*, so stores through its range reach the file.
- **`VerifyPath`** — create every missing directory along a path.

## Notes

`IReaderBase` exists so that a second reader implementation — the windowed streaming reader in [`stream_reader.h`](stream_reader.h.md), which never holds the whole file — gets the same read vocabulary without inheriting the pointer-into-a-buffer contract. In a rebuild this is one trait or interface with two implementations; the C++ shape (a template parameterized on the implementation, to keep the reads non-virtual) is incidental.
