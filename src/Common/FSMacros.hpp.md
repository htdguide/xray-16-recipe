# src/Common/FSMacros.hpp

> The fixed names of the logical roots the virtual filesystem resolves paths against, and the path separator the engine writes internally.

**Needs** — _(none)_
**Used by** — [`Common.hpp`](Common.hpp.md) · [`GameMtlLib.cpp`](../xrMaterialSystem/GameMtlLib.cpp.md)
**Tier floor** — T4: a table of string constants. Nothing about it constrains the language.

## Purpose

Every path the engine handles is written relative to a *logical root* — a name in dollar
signs that the filesystem configuration maps onto a real directory or archive. The names
are part of the shipped data: the game's configuration files, level directories and tool
outputs all spell them, so they cannot be renamed. This file is where the ones the C++ side
refers to by symbol are spelled once.

## State

```text
RECORD LogicalRoots                 # frozen: these spellings appear in shipped data files
  # game roots, used by the running engine
  game_data       : text = "$game_data$"
  game_textures   : text = "$game_textures$"
  game_levels     : text = "$game_levels$"
  game_sounds     : text = "$game_sounds$"
  game_meshes     : text = "$game_meshes$"
  game_shaders    : text = "$game_shaders$"
  game_config     : text = "$game_config$"

  # editor and tool roots, used only by the asset pipeline
  server_root      : text = "$server_root$"
  server_data_root : text = "$server_data_root$"
  local_root       : text = "$local_root$"
  import           : text = "$import$"
  sounds           : text = "$sounds$"
  textures         : text = "$textures$"
  objects          : text = "$objects$"
  maps             : text = "$maps$"
  temp             : text = "$temp$"
  detail_objects   : text = "$detail_objects$"
  omotion          : text = "$omotion$"     # one authored animation
  omotions         : text = "$omotions$"    # a bank of shared animations
  smotion          : text = "$smotion$"     # one skeletal animation
```

**Invariants** — a logical root is a whole path segment: it is matched as a prefix ending at
the separator, never as a substring. The full set of roots is larger than this list — the
filesystem configuration file defines roughly twenty more — and this file names only the
ones reached from code rather than from data.

## Path separator

**Contract** — the engine's internal spelling of a path uses the backslash as separator. That
is a consequence of the game data having been authored on Windows: archive directories and
configuration files contain backslashes, so the engine adopts them as its canonical form
and converts to the host's separator at the moment it touches a real file.

**Notes** — two constants exist for one character, one for scanning a path and one for building
one, purely because C++ distinguishes a character literal from a string literal. That
distinction carries no decision. What carries a decision is the direction of conversion:
canonical form is the *data's* form, and the host's form is the exception applied at the
boundary. The conversion functions themselves live in the per-platform headers —
see [`PlatformLinux.inl`](PlatformLinux.inl.md).
