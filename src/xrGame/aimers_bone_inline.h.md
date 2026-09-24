# src/xrGame/aimers_bone_inline.h

> Distributes one aiming correction over a chain of bones, so a character turns to face a target with its whole spine rather than snapping one joint.

**Needs** — [`aimers_bone.h`](aimers_bone.h.md) · [`aimers_base.h`](aimers_base.h.md)
**Used by** — [`aimers_bone.h`](aimers_bone.h.md)
**Tier floor** — T2: recursive geometry driving the skeleton evaluator.

## Purpose

[`aimers_base.cpp`](aimers_base.cpp.md) solves one bone's rotation. A single bone rotating
far enough to aim looks broken — a neck does not turn ninety degrees. This file spreads the
correction down a chain: each link takes a share, the next link re-solves against the
already-corrected pose, and the residual shrinks as the chain is consumed.

## State

```text
RECORD ChainAimer               # extends the aimer base
  bone_ids : list<int>          # the chain's bones, base first, tip last
  bones    : list<transform>    # their sampled world poses, refreshed per link
  result   : list<transform>    # the correction for each link, in object space

# Invariant: result[i] is expressed in the object's own frame, not the world's —
#   every solved rotation is conjugated by the reference transform before being
#   stored, so it can be handed to the skeleton, which works in object space.
```

## Construction

**Contract** — Resolves the bone names to skeleton indices and immediately solves the whole
chain. The correction for every link is available as soon as the object exists.

## `compute_bones` — the distribution

**Contract** — Solves link *i*, converts its rotation into object space, scales it down to
its share, installs it on the skeleton as a temporary hook, and recurses to link *i+1* —
which therefore samples a pose that already includes every previous link's share. Restores
each hook on the way back out.

```text
FUNCTION compute_bones(i)
  compute_bone(i)                                   # solve this link in world space
  result[i] = inverse(reference) . result[i] . reference   # express it in object space
  IF i is the last link THEN RETURN

  # Take only a share of the rotation. The divisor counts the links *after* this one,
  # so the first of three links takes a half, the second takes all of what remains.
  angles = euler angles of result[i]
  angles = angles / (chain_length - i - 1)
  result[i] = rotation from angles

  install result[i] as link i's skeleton hook
  compute_bones(i + 1)          # the next link now solves against the corrected pose
  restore link i's previous hook
```

**Invariants** — The hook must be installed *before* the recursion and restored *after* it,
because the recursive call samples bone poses through the skeleton and must see the
partial correction. The restore is what keeps the aimer from leaving the character
permanently bent: an aimer is a query, and the caller decides separately whether to apply
the result.

The share is taken on Euler angles rather than by interpolating the rotation properly. That
is an approximation and it is visible: for large corrections the decomposition is not
uniform and the chain's shares do not sum to the original rotation. It is accepted because
the residual is re-solved at every link, so an inexact share is corrected by the next one
rather than accumulating. A rebuild may divide the rotation properly and will get a
slightly different, smoother distribution.

The divisor gives the *last* link a divisor of one — it takes the entire residual. The
chain therefore ends exact regardless of how the earlier shares were approximated.

## `compute_bone` — solving one link

**Contract** — Samples the link's pose and the chain tip's pose under the candidate
animation, then aims: the link is the pivot, and the thing being aimed is the *tip of the
chain*, using the tip's position and its forward axis as the ray.

```text
FUNCTION compute_bone(i)
  sample bone i and the chain tip under the candidate motion
  IF i is not the tip
    result[i] = aim_at_position(pivot = bone[i].position,
                                ray_origin = tip.position,
                                ray_direction = tip.forward_axis)
  ELSE
    result[i] = identity
```

**Notes** — The tip's own correction is the identity, and the direct aiming code that would
have rotated it to face the target is present but disabled. The consequence is that the
last link never contributes: a three-bone chain effectively aims with two. Whether that is
a deliberate "do not rotate the head itself" decision or an abandoned experiment is not
recoverable from the source — the disabled branch is the straightforward
rotate-to-face-target, so it was tried and rejected.

Sampling asks for the link's pose and the tip's pose in one call, because sampling is the
expensive part — it plays the whole animation on a scratch channel — and doing it once per
link is already the cost of this design.
