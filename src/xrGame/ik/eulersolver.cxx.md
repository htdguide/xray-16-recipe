# src/xrGame/ik/eulersolver.cxx

> A rotation matrix, under a fixed three-axis convention, has exactly five entries that
> matter: one gives a joint angle's sine directly, four give the other two joints as
> quadrant-correct ratios. This file is the table of where those five entries are for each
> convention, and the two things you can do with them.

**Needs** — [`eulersolver.h`](eulersolver.h.md) · [`jtlimits.h`](jtlimits.h.md) · [`math3d.h`](math3d.h.md)
**Used by** — reached through its declarations in [`eulersolver.h`](eulersolver.h.md); callers name that, not this file.
**Tier floor** — T2. Table lookup and scalar trigonometry.

## Purpose

The chain solve produces two rotation matrices — one for the hip, one for the ankle — and
the skeleton needs six joint angles. This file performs that conversion, and it performs
it in two different modes which a rebuilder should keep apart in their head:

- **Given a matrix**, give me the angles. This runs every frame for every leg and is the
  only mode the shipping engine uses.
- **Given the matrix as a function of the swivel angle**, give me the ranges of swivel
  angle for which the angles stay legal. This runs only behind the joint-limits flag.

## State

```text
RECORD ConventionEntry          # one row per Euler convention; a fixed table of four
  simple_joint     : int        # which of the three angles is read directly
  simple_entry     : (row, col, sign)
  complex1         : int
  complex1_sin     : (row, col, sign)     # the entry holding sin(theta)*cos(simple)
  complex1_cos     : (row, col, sign)     # the entry holding cos(theta)*cos(simple)
  complex2         : int
  complex2_sin     : (row, col, sign)
  complex2_cos     : (row, col, sign)
  form             : {sine, cosine}       # whether simple_entry is a sine or a cosine
```

Every row in the shipped table has the middle angle as the simple joint and the sine form.
The cosine form is representable and is implemented nowhere: every path that meets it
prints a complaint and returns nothing.

**Invariant** — the table is indexed by the convention's numeric value. The enumeration's
numbering is therefore part of the data, not a detail, and the range check that guards the
lookup terminates the process on a bad index rather than returning an error. A rebuild
should key the table by a closed enumeration and delete both the check and the exit.

```text
RECORD SwivelDecomposer
  convention : ConventionEntry
  simple     : SimpleJoint        # from jtlimits
  complex1   : ComplexJoint
  complex2   : ComplexJoint
  singular   : list<real>         # the shared singular swivel angles, at most 2
```

## Why five entries

Composing three axis rotations, the resulting matrix has one entry that is `±sin` of the
middle angle alone — every other term cancels — and four entries of the form
`±sin(outer)·cos(middle)` and `±cos(outer)·cos(middle)`, two for each outer angle. So:

```text
sin(middle)              =  ±R[a][b]
sin(outer1)*cos(middle)  =  ±R[c][d]
cos(outer1)*cos(middle)  =  ±R[e][f]
sin(outer2)*cos(middle)  =  ±R[g][h]
cos(outer2)*cos(middle)  =  ±R[i][j]
```

The middle angle comes from an inverse sine — two answers, a half turn apart in effect —
and each outer angle from a quadrant-correct arctangent of its pair, in which the common
`cos(middle)` cancels. **Family one is the branch where that cosine is positive; family
two is where it is negative**, and in family two both arctangent arguments are negated,
which moves both outer angles by a half turn. That is the entire meaning of "two
families", and it is why they come in a matched set: you cannot take the middle angle from
one and an outer angle from the other.

## `EulerSolve` · `EulerSolve2`

**Contract** — decompose a rotation matrix into three angles under a named convention. The
first picks a family; the second returns both, and derives the second family from the
first by arithmetic rather than by recomputing it.

```text
FUNCTION decompose(convention, R, family) -> (a0, a1, a2)
  e <- table[convention]
  v      <- e.simple_entry   read out of R with its sign
  y1, x1 <- e.complex1 pair  read out of R with their signs
  y2, x2 <- e.complex2 pair  read out of R with their signs

  IF family is 1                        # cos(middle) > 0
    angles[e.simple_joint] <- arcsin restricted to the quarter turns either side of zero, of v
    angles[e.complex1]     <- normalize(arctan2( y1,  x1))
    angles[e.complex2]     <- normalize(arctan2( y2,  x2))
  ELSE                                  # cos(middle) < 0: negate both ratios
    angles[e.simple_joint] <- arcsin restricted to the other half, of v
    angles[e.complex1]     <- normalize(arctan2(-y1, -x1))
    angles[e.complex2]     <- normalize(arctan2(-y2, -x2))
  RETURN angles

FUNCTION decompose_both(convention, R) -> (family1, family2)
  family1 <- decompose(convention, R, 1)
  # The second family is the first, reflected: the middle angle about a quarter turn,
  # the two outer angles by a half turn each.
  family2 <- ( half_turn - family1.middle, family1.outer1 + half_turn,
                                            family1.outer2 + half_turn ), all normalized
```

**Notes** — the second family is computed by formula rather than by a second inverse-sine
call, which is both faster and exactly consistent. A rebuild that computes it
independently will occasionally produce a pair that disagrees in the last bit and will
then see the family-selection logic in [`limb.cxx`](limb.cxx.md) flip between frames.

Note that the entries carry a sign in the table rather than being negated in code. That is
what lets one routine serve four conventions; the sign is the only thing that differs
between the mirrored left and right variants.

## `EulerEval`

**Contract** — the inverse: compose three angles into a rotation matrix under a named
convention. Each convention names three axes *and* three signs — two of the four flip the
sense of one or two of the rotations — and the composition is in the stated order.

**Notes** — this is the forward-kinematics side and exists mainly so that a rebuild can
check the decomposition by round-tripping. The engine's own forward pass goes through
[`limb.cxx`](limb.cxx.md) instead.

## `SwivelDecomposer` — construction

**Contract** — built once per goal, from three matrices `C`, `S`, `O` such that the
rotation is `cos ψ·C + sin ψ·S + O`, plus the three joints' low and high limits. It reads
the same five table entries, but out of the three coefficient matrices rather than out of
a single rotation, giving each entry as a sinusoid in ψ. Those become one simple joint and
two complex joints from [`jtlimits`](jtlimits.cxx.md). The singular swivel angles are
computed once, from the first complex joint, and shared with the second.

```text
FUNCTION build(convention, C, S, O, low[3], high[3]) -> SwivelDecomposer
  e <- table[convention]
  # Each matrix entry, as a function of psi, is alpha*cos(psi) + beta*sin(psi) + xi
  # with the three coefficients taken from the same cell of C, S and O. The table's
  # sign is applied to all three together.
  simple   <- SimpleJoint( curve at e.simple_entry, limits of e.simple_joint )
  complex1 <- ComplexJoint( curves at e.complex1 pair, curve at e.simple_entry,
                            limits of e.complex1 )
  complex2 <- ComplexJoint( curves at e.complex2 pair, curve at e.simple_entry,
                            limits of e.complex2 )
  singular <- complex1.singularities()     # shared: both complex joints have the same
                                           # vanishing factor, and cutting the circle at
                                           # two slightly different places would produce
                                           # arcs that fail to merge
```

## `SolvePsiRanges`

**Contract** — for each of the three joints and each of the two families, the set of
swivel angles that respect that joint's limits. Six sets in all, delivered as two arrays
of three. The caller intersects them. Costs the three joints' own analyses and nothing
more.

## `Solve` · `Derivatives` · `Singularities`

**Contract** — the three joint angles for a given swivel angle and family; their
derivatives with respect to the swivel angle; and the shared singular swivel angles. A
further overload decomposes a matrix directly, identical to the free function above.

**Notes** — the angles are written into the caller's array **by joint index from the
table**, not in the order the three sub-solvers are asked. That indirection is what lets
the caller work in skeleton-joint order while the table works in convention order, and it
is the second place — after the family pairing — where a rebuild's off-by-one will produce
a pose that is wrong in a way that still looks like a leg.
