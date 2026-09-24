# src/xrGame/stalker_animation_data_storage.cpp

> Shares one loaded animation table between every stalker whose model draws on the same motion banks, keyed by the bank list rather than by the model.

**Needs** — [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md) · [`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md) · [`stalker_animation_data.h`](stalker_animation_data.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — reached through its declarations in [`stalker_animation_data_storage.h`](stalker_animation_data_storage.h.md); callers name that, not this file.
**Tier floor** — T3: a small linear cache with a structural key

## Purpose

Resolving a stalker's animation table means several hundred name lookups in a motion bank.
The shipped games have dozens of stalker models that share the same handful of banks, so
doing it per model would repeat nearly all of that work. This cache does it per *bank list*
instead.

## State

```text
RECORD AnimationDataStorage
  entries : list<(skeleton, AnimationData)>   # the skeleton is kept only as a key witness
```

**Invariants** — the stored skeleton is a *representative*, not an owner: it is the first
skeleton that asked for this entry, kept so that a later query can compare bank lists
against it. Entries are never invalidated, so that skeleton must outlive the storage, which
it does because the storage is cleared on level unload.

The table is owned by the storage and destroyed with it. The stalkers that hold it hold it
by reference and must not outlive it — the same level-unload ordering.

## `object`

**Contract** — return the animation table for a skeleton, loading it on first ask for a
given bank list. Linear search; the table is small. Blocks on a miss (the load).

```text
FUNCTION object(skeleton) -> AnimationData
  FOR EACH (witness, data) IN entries
    IF same_banks(skeleton, witness)  RETURN data
  data = load a new AnimationData from skeleton
  APPEND (skeleton, data) TO entries
  RETURN data

FUNCTION same_banks(a, b) -> bool
  IF a.bank_count != b.bank_count  RETURN false
  FOR i IN 0 .. bank_count - 1
    IF a.bank(i) != b.bank(i)  RETURN false     # identity of the bank, not its name
  RETURN true
```

**Invariants** — the key is the **ordered list of motion banks**, compared by bank identity.
That is the right key rather than the model name, because two different stalker models
loading the same banks resolve every motion to the same result — the tables would be
identical — while one model loaded with a different bank set would not. Order matters
because a motion name is resolved by searching the banks in order and the first hit wins, so
two models with the same banks in a different order can genuinely disagree.

Comparing bank identity rather than bank name is what makes the comparison cheap enough to
run linearly over the whole table on every stalker's reinitialization.

**Notes** — a linear scan is right here: the number of distinct bank lists on a level is a
handful, and a hash over an ordered list of handles would cost more to compute than the scan
costs to run.

## `clear`

**Contract** — destroy every loaded table. Called on level unload, and from the destructor.
Every stalker holding a table must already be gone.

## The storage itself

**Contract** — a single process-wide instance, created on first use and never recreated. See
[`stalker_animation_data_storage_inline.h`](stalker_animation_data_storage_inline.h.md).
