# Schedule expressions + cross-stratum delta tracking — design

Companion to the egglog-on-Souffle research project. This is a fork of
Souffle 2.5 on branch `bounded-iteration`.

## Problem

Egglog programs use a nested-fixpoint schedule:

```
(saturate
  (seq user_rules (saturate rebuild)))
```

Souffle's stratified evaluation never re-visits a stratum, so deltas reset
to empty between visits. To express this schedule efficiently we need
two things together:

1. **Schedule expressions** — let the user compose strata (sequence,
   saturate, repeat).
2. **Cross-stratum delta tracking** — when a stratum re-enters within an
   outer saturate, its δ must be *the change since its last visit*, not
   empty (which would lose semi-naive) and not the full relation (which
   would duplicate work).

Without (2), schedule expressions are correct but `O(|relation|)` per
outer iteration — same cost as external driving.

## Approach: snapshots at re-entry

When a stratum exits, snapshot every relation it can re-derive against.
On re-entry, the initial δ is `current ∖ snapshot`.

### MVP scope

Defer the full schedule-expression DSL. v0 is:

- One new program-level directive: `.outer_saturate` — wraps the entire
  SCC sequence in an outer fixpoint loop.
- Each recursive SCC tracks snapshots of its main relations across
  outer-loop iterations.
- Non-recursive SCCs are re-run on each outer iteration (correct but
  wasteful — fix in v1).
- Outer loop exits when no SCC produces δ this iteration.

This validates the snapshot mechanism on the core question: "does
cross-stratum delta tracking preserve semi-naive correctness while
allowing re-visits?" If yes, we add named strata + schedule expressions
in v1.

### Mechanism per recursive SCC

For each main relation `R` of the SCC, allocate a snapshot relation
`@snap_R`. Initially empty.

On stratum entry within the outer loop:
```
@delta_R := R ∖ @snap_R    // tuples added by other strata since last visit
                          // Initial entry: snapshot is empty, so delta = R (correct).
@new_R := empty
```

Run the fixpoint loop normally — non-recursive rules generate further
tuples, semi-naive proceeds with the seeded δ.

On stratum exit (after fixpoint):
```
@snap_R := R              // refresh snapshot for next outer iteration
```

The standard `Preamble` step that primes δ from main needs replacement;
its current behavior `δ := main` is wrong on re-entry. Replace with the
diff above.

### Outer-loop exit condition

```
Exit (∀ SCC s, ∀ relation R in s, |@new_R after first iteration| == 0)
```

Simpler approximation: track a single "anything changed this iteration"
boolean across all strata. If false at end of one outer iteration, exit.

### What snapshots cost

Memory: one extra btree per main relation per SCC, peaking at the size
of that relation. For the egglog encoded program, the largest relations
are the view tables (`AddView`, `MulView`) and UF tables. Doubling these
is the main cost.

CPU: snapshot refresh = one set-copy per SCC exit. Set-difference =
one negation-scan per SCC entry. Both are linear in relation size.
Compared to a full re-evaluation per outer iteration, this is a win
whenever the per-iteration delta is small relative to the relation.

## Restrictions in v0

1. **Positive rules only.** Negation is sound only if subsumption-style
   monotonicity is preserved across visits. We document this and check
   programs at parse time.
2. **No nested schedule expressions yet.** v0 has one fixed schedule:
   outer-saturate over topological SCC sequence. Named strata + DSL come
   in v1.
3. **Non-recursive SCCs re-run fully each outer iteration.** Correct but
   wasteful. Fix in v1 by handling non-recursive snapshots too.
4. **Subsumption interaction undocumented.** Subsumption deletes from R;
   `R ∖ snap` only captures additions. For pure-additive rules this is
   correct (subsumption preserves logical content). For rules whose
   correctness depends on tuples being *deleted* (e.g., negation-based
   live views), this is unsound. Egglog's encoded program uses
   subsumption only for canonicalization, where additive deltas are
   correct, so v0 should be fine for our use case.

## Implementation plan

### Files to change

| File | Change | LOC |
|---|---|---|
| `parser/scanner.ll` | new token `.outer_saturate` | ~1 |
| `parser/parser.yy` | parse top-level directive into program flag | ~10 |
| `ast/Program.{h,cpp}` | new bool flag `outerSaturate` | ~10 |
| `ast2ram/utility/TranslatorContext.{h,cpp}` | expose flag | ~5 |
| `ram/Snapshot.h` (new) | new RAM op? Or reuse existing relation ops | ~50 |
| `ast2ram/seminaive/UnitTranslator.cpp` | snapshot ops at SCC boundaries; outer Loop | ~150 |
| `interpreter/Engine.cpp` | dispatch new RAM ops | ~30 |
| `synthesiser/Synthesiser.cpp` | C++ codegen for new RAM ops | ~30 |

Estimated 250–350 lines total for MVP (less than the 1500–2500 I quoted
earlier — we're using existing relation ops, not changing the storage
layer).

### Phasing

1. `.outer_saturate` flag plumbed through parser → ast → context
2. Outer Loop wrapping at end of `generateProgram` — without snapshots,
   to test that the structure works (will produce wrong answers, but
   compiles + runs)
3. Snapshot relations allocated alongside main relations
4. Snapshot init/diff/refresh ops at SCC boundaries
5. Outer-loop exit condition
6. Test on a small egglog-style benchmark

### Test program

```souffle
.type Math = [tag: number, a: Math, b: Math]
.decl AddView(a: Math, b: Math, leader: Math)
.decl UF(child: Math, parent: Math)
.decl Term(t: Math)

// (Source rules omitted — same as q4-fresh-ids-4 + a divergent
// distributive rule bounded by .limititerations)

.outer_saturate
```

Success criterion: the program runs to convergence, produces the same
result as without `.outer_saturate`, but with measurably less work
than re-evaluating from scratch each outer iteration.

## What v1 adds

If MVP works:

- `.stratum NAME { ... }` for manually grouping rules into named strata
- `.schedule (saturate (seq NAME (saturate NAME)))` DSL
- Per-stratum snapshots (rather than per-program)
- Better integration with `.limititerations` for `(run N)` semantics
- Non-recursive SCC snapshot handling
