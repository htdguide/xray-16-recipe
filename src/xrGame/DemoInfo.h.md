# src/xrGame/DemoInfo.h

> Declares the demo summary record and its per-player line, implemented in [`DemoInfo.cpp`](DemoInfo.cpp.md).

**Needs** — [`DemoInfo.cpp`](DemoInfo.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`DemoInfo.cpp`](DemoInfo.cpp.md) · [`DemoInfo_Loader.cpp`](DemoInfo_Loader.cpp.md) · [`DemoInfo_Loader.h`](DemoInfo_Loader.h.md) · [`Level_network_Demo.cpp`](Level_network_Demo.cpp.md) · [`MainMenu.cpp`](MainMenu.cpp.md) · [`MainMenu.h`](MainMenu.h.md) · [`UIDemoPlayControl.cpp`](ui/UIDemoPlayControl.cpp.md)
**Tier floor** — T3: a declaration plus two size bounds

## Purpose

Declares the two summary records, their three directions (from the live match, to a file,
from a file) and the size bounds that make the format readable without trusting it.
Substance in [`DemoInfo.cpp`](DemoInfo.cpp.md).

The bounds are the part a rebuild needs from the header itself:

```text
string_max      = 256 bytes                     # per string
player_max      = string_max + 80               # per player record
summary_max     = player_max * max_players
                + string_max * 5 + 4            # the whole summary
```

Exported units:

- `demo_player_info` — one scoreboard line: name, frags, deaths, artefacts, derived score,
  team, rank. Non-copyable, because the summary owns its lines by reference.
- `read_from_file`, `write_to_file`, `load_from_player` — the three directions.
- `demo_info` — map name and version, mode name, final score, author, and the owned list
  of lines.
- `read_from_file`, `write_to_file`, `load_from_game` — the three directions.
- `sort_players` — reorder in place by a caller-supplied comparator; the record imposes no
  ordering of its own.
- `get_player`, `get_players_count` and the field accessors — read-only.
- Script registration for both records: accessors only, no constructor.
