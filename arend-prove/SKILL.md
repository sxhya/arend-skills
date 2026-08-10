---
name: arend-prove
description: Procedural workflow for filling in `{?}` placeholders in an Arend file once the statement already typechecks, PLUS the catalog of metas (`rewrite`, `run`, `in`, `at`, `cases`, `mcases`, `simplify`, `equation`, `cong`, `ext`, `linarith`, `unfold`, `assumption`, `using`, `later`) you'll reach for there. Use whenever an .ard file is at the "only GOAL errors remain" stage, or whenever it imports from `Paths.Meta` / `Meta` / `Algebra.Meta` / `Function.Meta` / `Logic.Meta`. Complement to **arend-formalize** (which gets you to "only GOAL"). Covers the proof-attempt feedback loop, when to decompose vs. attempt directly, the meta semantics and a decision table, mechanical gotchas, and the lookup discipline (`-ss`, `-ps`, `-fu`, `-ch`).
user-invocable: true
---

# Proving in Arend — proof loop and meta catalog

Picks up from **arend-formalize** ("statement typechecks, only `{?}` remain"). Covers the inner loop of closing those goals AND the catalog of metas you'll reach for. Companion to **arend-quirks** (language paper-cuts). Write-time style and compaction idioms are summarized in *Style — compaction* below and in **arend-formalize**; they are the target the *first* time, not just on a post-pass.

A meta is surface syntax that elaborates to ordinary kernel terms (transport, pmap, etc.). Failures look like normal type errors, but the *cause* is usually one of: wrong imports, `\lemma` vs `\func` mismatch, or asking the meta to do more than it actually can — see *Common failure modes* below.

---

## Before anything: pass `--serialize`

Closing goals is iterative — you'll typecheck dozens of times. Don't pay the cold-load cost each round:

```bash
cd /path/to/arend-lib && arend --serialize <Module>:<def>
```

**Serialization is opt-in** (commit `9a59b3151`). Without `--serialize` no `.arc` binary caches are written, so every one of those dozens of rounds re-typechecks arend-lib from source (~2 min) instead of deserializing the unchanged cone and re-checking only your edit (seconds). Loading existing `.arc` files needs no flag — it's on by default; `--serialize` controls the *write* side only. Everything below assumes you're passing it.

**Turn it off only while diagnosing the cache.** If you hit phantom errors on definitions you didn't touch, bogus mismatches naming the same type twice (`Rat` vs `Arith.Rat.Rat`), or a stack trace out of the (de)serializer, re-run as `arend -r --serialize <Module>` — `-r` skips *loading* the caches but does not delete them, so bare `-r` leaves the poisoned `.arc` on disk for the next plain run to pick up again. Pairing the two overwrites them from a clean build. Go straight back to `--serialize` afterwards.

---

## The inner loop

For each `{?}` you intend to attack:

1. **Read its full `[GOAL]` block.** Both the expected type *and* the binder list (`Context: ...`). For sub-goals inside `\have` / `\case` chains, the binder list is what tells you what's in scope. Scope the run with the positional rather than filtering the output: `arend --serialize <Module>` lists every goal and error in that module, `arend --serialize <Module>:<def>` narrows to a single definition.
2. **Decide: prove, or decompose?** (See next section.)
3. `arend --serialize <Module>:<def>`. Read the first error or remaining GOAL. Iterate.

---

## Decomposition is the main move

Don't write 200 lines of proof in one expression. Any goal that takes more than one mathematical step:

- Name each step as its own helper lemma with a clean type.
- Replace the original `{?}` with `combine(helper1, helper2, ...)`.
- Leave any helper as `{?}` if it's hard — the structure now typechecks, the difficulty is isolated, and downstream proofs can use the helper as a black box.

A `{?}`'d helper with the right type is more valuable than a half-finished inline proof: it commits you to the proof's structure and lets the rest of the file move forward. It also lets you split work across sessions — name the hard helper, prove the easy half, come back to the hard half later.

> Heuristic: if you can't write the proof in one breath, name the next sub-statement as a helper and continue.

---

## Upstream missing prerequisites

If you find yourself wanting a 1–5 line helper that should exist (`partialSum_suc`, `BigSum_assoc`, monotonicity-of-X), go add it to the right module instead of working around its absence. Working around is typically 10× the size of the right helper, and you'll keep wanting it across multiple proofs.

**Do not log missing helpers/statements in `arend-issues.md`.** A gap in arend-lib is not an "issue" — it's a contribution opportunity. Just add the helper to the right module and move on. `arend-issues.md` is reserved for tooling friction (typechecker quirks, slow CLI paths, misleading error messages), not library gaps.

---

## Imports — what comes from where

| Need | Import |
|---|---|
| `idp`, `pmap`, `transport`, `transportInv`, `inv`, `*>`, `coe` | `Paths` |
| `rewrite`, `rewriteI`, `ext`, `simp_coe` | `Paths.Meta` |
| `run`, `in`, `at`, `cases`, `mcases`, `unfold`, `assumption`, `later` | `Meta` |
| `$`, `#`, `repeat`, `__` (anonymous-section placeholder) | `Function.Meta` |
| `equation`, `cong`, `linarith`, `simplify` | `Algebra.Meta` |
| `Empty`, `Not`, `&&`, `||`, `propExt` (the type) | `Logic` |
| `contradiction`, `cases`-style logic helpers | `Logic.Meta` |
| `Pointed`, `Monoid`, `Group`, `Ring`, `Semiring` classes | `Algebra.{Pointed,Monoid,Group,Ring,Semiring}` |
| `LinearlyOrderedSemiring`, `OrderedRing` (the classes `linarith` dispatches on) | `Algebra.StrictlyOrdered` and/or `Algebra.Ordered` |
| `Nat` as a `LinearlyOrderedSemiring` instance, `<`, `<=` on `Nat` | `Arith.Nat` |

**Boilerplate that almost always works for an algebraic-meta file:**

```arend
\import Algebra.Meta
\import Algebra.Group
\import Algebra.Monoid
\import Algebra.Pointed
\import Algebra.Ring
\import Algebra.Semiring
\import Logic
\import Logic.Meta
\import Meta
\import Paths
\import Paths.Meta
```

For an equality-meta file (no algebra needed):

```arend
\import Paths
\import Paths.Meta
\import Meta
\import Function.Meta
\import Logic
```

The names `Algebra.Meta`, `Paths.Meta`, `Meta`, `Function.Meta` are *virtual* (provided by the Java extension `org.arend.lib.StdExtension`) — they don't appear as files in `arend-lib/src/`, but they import fine. Don't grep for them.

---

## Common failure modes (read this first)

### Meta- and goal-language failure modes

1. **`\lemma` requires the type to be a proposition.** `f x = f y` for `f : A -> B` with `B : \Type` is *not* a Prop — it lives in `\Type`. The error is `The type of a lemma must be a proposition`. Default to `\func`; promote to `\lemma` only after a clean typecheck reports a warning.

2. **`simplify` is not a decision procedure.** It strips identity elements, double inverses, zeros — that's it. It does *not* do commutativity or distribution, and it can fail even when both sides should normalize to the same expression (e.g. `x * x = (x * (ide * x)) * ide` fails on a `Monoid`). When `simplify` fails, fall back to `equation`, manual `rewrite`, or `pmap` chains.

3. **Imports are unforgiving.** Every meta lives in a specific module. Missing `Logic` → `Empty` is unresolved. Missing `Algebra.Group` → `negative` is unresolved. The virtual modules (`Algebra.Meta`, `Paths.Meta`, `Meta`, `Function.Meta`) import fine but don't appear in `arend-lib/src/`.

4. **`Nat.<=` does not parse.** `Nat` is a `\Set0`, not a class. Use bare `<=` after `\import Arith.Nat`. The arithmetic operators `Nat.+` and `Nat.*` *do* parse — those are special syntactic sugar resolved by the language, not class-member access.

5. **`Empty` from `Logic` collides with a local `\data Empty`.** If `Logic` is in scope, drop the local `\data Empty`. Mixed scope = "ambiguous reference".

6. **`run { f1, f2, ..., t }` puts the seed at the *end*, not the start.** `run { rewrite p, rewrite q, idp }` means `rewrite p (rewrite q idp)`. If you put `idp` first, you've inverted the chain.

7. **`rewrite p in t` ≠ `rewrite p t`.** `in` strips the expected type and uses `t`'s type instead. For *most* metas this is invisible. For `rewrite`, which inspects the expected type to find the LHS, the difference is observable. If `rewrite p q` complains about not finding the LHS, try `rewrite p in q`.

8. **`Records.ard` from `tutorial-code-fresh` (`/home/sergey/Documents/tutorial-code-fresh/PartI/src/Records.ard`) defines its own `Monoid`/`Ring`/...** It will shadow arend-lib's classes — `linarith` and friends won't see the right instances. Either don't `\import Records` in a metas file, or rename the local classes.

9. **A meta inside `run { ... }` with a mysterious type error usually wants `later $`.** `later` (from `Meta`) defers a meta invocation until type inference has resolved more of the expected type. When the intermediate goal at some position in a `run` chain isn't pinned down enough for the meta to fire, prefix with `later $`: `run { later $ rewrite p, idp }`. Rule of thumb: if a meta in the middle of a `run` complains about an unresolved goal but the same meta works on its own, try `later $`.

10. **`equation` defaults to the non-commutative ring solver.** If you call plain `equation` on an identity that needs `*`-commutativity (e.g. anything over `Rat`, `Real`, or any concrete commutative ring), it fails with `Ring solver failed` and the printout shows the two sides differing only by argument order — `v0*v1*v2` vs `v2*v0*v1`. The error message doesn't suggest the fix. Use `equation.cRing` for commutative rings, `equation.cMonoid` for commutative monoids, `equation.monoid` for non-commutative monoids. Established practice in `Algebra/Ring/Localization.ard` and `Algebra/Linear/Matrix.ard`. When the structure is *abstract* (`{R : CRing}` etc.), `equation` may pick the right solver from the class — but for concrete arithmetic over `Rat`/`Real`/`Nat` you almost always need the explicit `.cRing` qualifier. **In-chain vs bare ascription:** the *standalone* `(equation.cRing : LHS = RHS)` over concrete `Rat` fails with `Cannot infer an instance of class 'CRing' with classifying expression: Rat` (nothing drives the instance), but the *same* `equation.cRing` embedded in a `*>` chain — `inv (equation.cRing *> pmap (…) lemma *> …)` — infers fine because the chain position pins its expected type. So when ascription fails, drop the ascription and let the chain pin the type rather than adding `{instance}` or a hand-written rearrange helper.

11. **`linarith` over `Real` finds its instance with just `\import Arith.Real.Field`** (since commit `ab3dc1dba`, which added `solveEquationsFor` to push a single inference variable). The old quirk — "needs `\open RealField`, instance resolution is unstable if multiple `linarith` calls coexist without the open" — no longer reproduces; verified with both implicit `{x : Real}` and explicit `(x y : Real)` parameters, with and without multiplication. If you still see `Cannot infer an instance of class 'LinearlyOrderedSemiring' with classifying expression: Real`, check the import is actually present; `\open RealField` is no longer required. (For `Rat`, `\import Arith.Rat` suffices for the same reason; for abstract `{R : LinearlyOrderedSemiring}` the instance was always pinned by the parameter.)

11b. **`linarith` does not distribute products of sums, and it is sensitive to monomial order.** Two limitations that together account for most "the printed assumptions clearly imply the goal, yet `Cannot solve the equation`" failures:

    - **A product one of whose factors is a non-constant sum is a single opaque atom.** `A * R * (1 - t) + eps * s` is linear once expanded, but `linarith` will not expand it — nor will it expand `(s + 1) * eps < q * (c - a0)`. Restate every such hypothesis in hand-distributed form and bridge with `<∘l =_<= equation.cRing` (or `=_<= equation.cRing <∘r (h <∘l =_<= equation.cRing)` when *both* sides need reshaping — `<∘l` only touches the right end).
    - **`eps * pow 3 n` and `pow 3 n * eps` are different atoms.** Two hypotheses that differ only in a product's factor order cannot be combined. Fix the order by hand across every hypothesis *and* the goal; the error output gives no hint that this is the problem, because the two spellings read as the same quantity.

    Also folding-related, same family:

    - **A `\func`-defined constant is not unfolded — but a `\meta` one is.** With `\func one/3 : Real => ratio 1 3`, the goal `pow one/3 n * eps <= one/3 * eps` carries no numeric content: `one/3` is an atom, and even `one/3 > 0` fails with `Cannot solve the equation`. Spelling out `(ratio 1 3 : Real)` at every site works but is verbose and leaves the file with two spellings of one constant. **The fix that keeps the abbreviation is `\meta`**, which is pure syntactic substitution, so `linarith` sees the literal:

      ```arend
      \private \meta one/3 => (ratio 1 3 : Real)                    -- linarith sees through this
      \private \lemma one/3>0 : one/3 > {RealField} 0 => linarith   -- now closes
      ```

      **The ascription is mandatory.** A bare `\meta one/3 => ratio 1 3` expands to a `Rat`, and every `pow one/3 k` then loses the instance that `one/3 : Real` used to pin — verified as ~12 `Type mismatch` / `Cannot find subexpression` errors across one file. Live use: `Arith/Complex/FTA/Kneser.ard:48`, where switching `\func` → `\meta` let all 14 workaround spellings of `ratio 1 3` collapse back to `one/3`.

      A *parameterised* `\func` such as `kneser-q n => 1 - pow one/3 (2 * n * n + n)` is a different case — it stays folded and is meant to (the named `kneser-q>0` / `kneser-q<1` lemmas are its interface). There, restate the goal with the body spelled out and convert at the end via `=_<= equation.cRing`.
    - **`Real.fromRat 1` is not the literal `1`** as far as the solver is concerned, even though they are definitionally equal. Re-annotating the hypothesis (`\have | h' : <same statement with 1> => h`) costs nothing and fixes it.
    - **Trim the context with `usingOnly`.** Past ~20 hypotheses the solver starts failing on goals it can do; and unrelated `Nat` facts in scope print as `coefMap {?this} (fromInt …)` with unresolved metas. `usingOnly (h1, …, h5) linarith` both speeds it up and makes the failure output readable.

    Worked example: `Arith/Complex/FTA/Kneser.ard:KneserLemma` (was `Arith/Real/KneserLemma.ard`, moved 2026-08-10) — `sums'`, `cond2'`, `q-eq`, `lower-scaled`, `eps-third` exist purely to hand the solver a distributed, order-normalised, `usingOnly`-restricted system.

12. **`linarith` chokes on symbolic identities — reach for `equation.cRing` instead.** `linarith` solves *linear (in)equalities* with atoms; it can normalize `2 * x + 3 * y <= 5` but not `(A - B) - (A + B) = -2 * B` where the goal is a symbolic identity (`2` may appear as `natCoef 2`, `negative (...)` blocks linarith's normalization). When linarith fails with `Cannot solve the equation` and prints two algebraically-equivalent sides (often differing in `natCoef 2` vs literal `2`, or by `negative` placement), switch to `equation.cRing`. Rule of thumb: linarith is for **numeric** inequality goals; `equation.cRing` is for **polynomial-identity** equality goals; they don't substitute for each other even when both seem applicable.

13. **`series_<=` needs its bound function to return `ExUpperReal`-typed values** with the `<=` resolved to `ExUpperRealAbMonoid.<=` (not generic `LinearOrder.<=`). When passing a bound to `series_<=`, write the result type with explicit `ExUpperRealAbMonoid.<=` qualifier, otherwise unification picks `LinearOrder.<=` and fails. The error reads `Cannot infer parameter 'b' of definition 'series_<='` — that's the symptom.

14. **An in-scope `RealStoneC*Algebra` (or any `StoneC*Algebra Real`) instance breaks `linarith` over `Real`.** `StoneC*Algebra` carries the C\*-positivity order (`0 <= a ⟺ ∃ y, a = y*y`). With that instance in scope (e.g. via a plain `\import Topology.StoneCStarAlgebra`), `linarith`/`=_<=` over `Real` resolve `<=`/`<` to the C\*-order instead of `RealField`'s linear order, and the goal explodes into `TruncP (\Sigma (y : E) (y * y = ...))` — Fourier–Motzkin can't run on it. The symptom is a `Type mismatch` whose *actual* type is that `TruncP (\Sigma (y : E) (y Semigroup.* y = ...))` form, on a `linarith` that combines hypotheses (trivial single-identity `linarith` often still passes, which is why `Arith/Log/Real.ard` survives — its goals are trivial). Merely being in scope makes it an instance-search candidate — `\open RealField` is irrelevant, and even a narrow `\import M (RealStoneC*Algebra)` still pollutes (it's still in scope).

    **Best fix — empty-import + local `\where \open`, no file split needed.** `\import Topology.StoneCStarAlgebra()` (empty parens) makes the module *referenceable for `\open`* while pulling **no** names into file scope, so the instance is not a global search candidate. Then, on each of the handful of definitions that actually need it, scope the name locally:
    ```arend
    \import Topology.StoneCStarAlgebra()          -- module referenceable, nothing in scope
    ...
    \sfunc arctan (x : Real) (|x|<1 : abs x < 1) : Real
      => arctan-gen x (real_<_U.1 |x|<1)
      \where \open Topology.StoneCStarAlgebra
    ```
    (Verbatim from `Arith/Trig/ArcTan.ard:47,110–112` as of 2026-07-28. A narrower `\open Topology.StoneCStarAlgebra(RealStoneC*Algebra)` works too; the file currently opens the whole module inside the `\where`, which is equally safe because the `\where` scope doesn't leak.)
    The whole `arctan` / `tan-arctan` development stays in one file (`Arith/Trig/ArcTan.ard`); the `linarith` proofs never see the instance. This is the idiom whenever you need a concrete `Real` Banach-algebra instance (for `arctan`, `log1+`-over-Real, etc.) in a file that also does `Real` field-arithmetic.

    *Heavier alternative (avoid unless the empty-import idiom can't apply):* mirror the `Arith/Log.ard` (generic) + `Arith/Log/Real.ard` (Real-specific) split — move the `RealStoneC*Algebra`-needing definitions into a base module that imports `StoneCStarAlgebra`, and import *that* base (not `StoneCStarAlgebra`) into the arithmetic file. Arend imports are non-transitive, so the instance stays out of the consumer's scope.

15. **`rewrite` searches the goal *syntactically*; it does not reduce the goal to expose a pattern.** Unification (`*>` chains, argument positions) normalizes, `rewrite`'s subexpression search does not. So a step whose LHS only appears *after* a reduction fails with `Cannot find subexpression`, even though the same lemma works fine as a chain element. Concretely, `BigSum (\new Array M 1 (f __))` reduces to `f 0 + 0` under unification, so `zro-right *> ide-right` typechecks against it — but `rewrite zro-right` reports

        Cannot find subexpression: {?error} + zro {{?error}}
           Expression: BigSum {…} (\new Array Complex 1 (\lam p0 => …)) + BigSum … = b 0

    Same for `pow x 0 → ide` and `pow z (suc k) → pow z k * z`. **Rule: when the pattern needs the goal to reduce first, write the step as a `*>`/`pmap` chain instead of a `rewrite`.**

16. **Concrete `Complex` (any record whose ring operations compute on fields) defeats both `equation.cRing` and `rewrite`.** `ComplexField.*` / `.+` unfold to `\new Complex (x.re * y.re - x.im * y.im) (…)` as soon as one operand is concrete, so:

    - `equation.cRing` on a `Complex` identity fails — the solver normalizes into `re`/`im` components and reports two unequal component-wise expressions (`v0 = …`, `v1 = …` full of `.re`/`.im`). Verified 2026-07-29: even `\lemma p {x y : Complex} : x * y = y * x => equation.cRing` fails. **Fix: state the algebraic step over an abstract `{C : CRing}` and instantiate it at `ComplexField`** — that is what the `cring-*` helpers in `Arith/Complex/FTA/Kneser.ard` (`opposite-direction \where`) exist for; they are load-bearing, not scaffolding. Give the helper a hypothesis (`(h : x * z = negative (e * y))`) so one lemma covers a whole rearrange-and-substitute step.
    - A meta *inside* a concrete-`Complex` operation is unsolvable: matching `cabs (?x * ?y)` against `cabs (b i * pow z i)` becomes higher-order once `*` unfolds, so `cabs_*`, `cabs_+`, `cabs_negative` need their implicits spelled out (`cabs_* {b i} {pow z i}`) — or, better, the operands must be *variables*, i.e. parameters of a helper lemma (`dir-cabs (b0 bk : Complex) (t : Real)`). Metas in **bare** positions are fine: in `cabs_+3 (X : Complex) {Y Z : Complex} … : cabs (X + (Y + Z)) <= cabs X + (y + z)`, `Y`/`Z` are inferred from the two bound arguments' types, but `X` had to be made explicit because its only other occurrence is under the `+`.

17. **`rewrite zro_*-left` / `zro_*-right` over `Real` can misfire.** `zro * x` normalizes to `fromRat zro`, so the pattern that gets searched is not the one you wrote and the rewrite can hit the wrong side (symptom: an argument's expected type collapses to `zro < fromRat zro`). Use `transportInv (\`< <the other side>) zro_*-left h` there — the explicit motive says exactly which occurrence to abstract.

### Workflow and language gotchas

- **`\peval` is for `\sfunc` only.** Plain `\func`s reduce by definition; `\peval` on them errors with `Expected a function or an \scase expression`.

- **`rewrite p` beats `transport (\lam x => x.<field> ...) p`.** The lambda-projection form fails on records with `Expected type: a class`. `rewrite` rewrites the goal directly without the lambda.

- **Inference failures want explicit pinning.** For `BigSum_<=` specifically, pass the length plus both arrays: `BigSum_<= {suc n} {arr1} {arr2} (\lam k => ...)`. More generally, when a lemma call gives `Cannot infer parameter X`, annotating one or two implicits is usually enough.

- **Use named primitives for sentinels.** `0 < 1` for rationals is `RatField.zro<ide`. Don't reconstruct them ad-hoc; the named form lets the elaborator unify with class instances cleanly.

- **`\have | x : T => v` can make `x` opaque.** With a type annotation, later uses of `x` may not unfold to `v` during unification (`Cannot solve equation: e, x`). If unification fails on a `\have`-bound name, either drop the type annotation or inline the value directly.

- **Don't over-qualify typeclass operators.** Inside expressions where both operands are typed (e.g. by surrounding `\have | x : Rat => ...` ascription), bare `+`, `*`, `<`, `<=` resolve to the right instance via type inference. `RatField.+` and `RatField.*` are usually redundant noise; reach for `Module.op` only when type inference can't pin the instance (rare). Named lemmas (`RatField.finv-left`, `RatField.zro<ide`) still need qualification because they aren't covered by the ambient `\open` set — a selective `\open RatField (finv, finv-left, ...)` shortens further if you reach for these often.

- **`\private` inside a `\where` block makes the helper unreachable from sibling lemmas.** The dot-path `parent.helper` syntax works for non-private `\where` members, but `\private` blocks it. The error is misleading: `Type mismatch: Expected type: a class. Actual type: <parent's body>`. Fix: drop `\private`, or hoist the helper to file-level `\private \lemma`.

- **Bridge `RealNormed.norm`-of-Rat to a pure Rat inequality** via `LatticeAbGroup.abs-ofPos` → `RealAbGroup.abs-rat` → `Real.<=-upper` → `rat_real_<=`. The chain unfolds `norm (Real.fromRat r) ≤ ExUpperReal.fromRat c` into `RatField.abs r ≤ c`. Live instances: `Arith/Log/Properties.ard:91` and `Topology/StoneCStarAlgebra.ard:251`; the shorter `norm_fromRat <=∘ ExUpperReal.<=-rat.1 …` variant is at `Arith/Trig/ArcTan.ard:95`. See **arend-quirks** §16.

- **For `\have`-bindings where `\elim` or case-splitting is needed**, factor into a separate `\private \lemma` rather than embedding `\case \elim` inside the `\have`. Arend's elaborator handles the recursion-via-named-lemma cleanly; embedded `\case` inside a `\have` chain often produces confusing errors. The original example (`ratio-recip>=0` / `ratio-recip<=1` in `Arith/Trig/ArcTan.ard`, where `\elim m` was needed to force `makeRat` reduction) has since been removed from the tree; for the underlying `makeRat` non-reduction and its current fixes see **arend-quirks** §17.

- **`Or` constructors `inl`/`inr` need `\import Data.Or`.** They live there, not in `Logic`. The error message (`Cannot find a reference to constructor among provided patterns`) doesn't say "missing import" — recognize the symptom.

- **After creating a new module, the symbol index may lag.** `-ss` is served from a per-library on-disk index that re-parses only changed files, so a brand-new module can be missing from `-ss` / `-fu` / `-ch` results for a round. A "no match" on something you just wrote is an index artefact, not evidence the definition is broken — typecheck the module, then query again.

- **Always pass `--serialize`.** Keeps the `.arc` caches current so the next round deserializes instead of re-typechecking from source. Default for every `arend` invocation, not just specific ones.

- **If `arend` hangs or comes back with empty output, suspect the binary caches.** Serialization round-tripping is still imperfect, so a stale or half-written `.arc` is the most likely cause. Re-run as `arend -r --serialize <Module>` — `-r` ignores the caches for this pass and `--serialize` replaces them from the clean build. (Bare `-r` skips loading without deleting anything, so the bad caches would just come back on the next plain run.)

- **A report that looks suspiciously short is a cache symptom, not a formatting one.** If the output disagrees with what's on disk, re-run with `-r --serialize` and compare — a from-source pass is the authoritative second opinion.

- **`arend <Module>` from outside the library crashes with `NullPointerException: ctx.outputRouter is null`.** Run the typecheck workflows and `-ss` / `-ps` / `-fu` / `-ch` / `-sc` from inside the library directory (`cd arend-lib && arend …`).

- **If `arend` throws a Java exception (stack trace, not a typecheck error), snapshot the cache state before recovering.** The current `.arc` binaries at the moment of crash are the only evidence for the serialization bug that produced them; recompiling from sources destroys that evidence. Before retrying, zip the entire arend-lib tree (sources, `bin/*.arc`, everything) to a timestamped archive outside the tree: `zip -qr /tmp/arend-lib-crash-$(date +%Y%m%d-%H%M%S).zip <path-to-arend-lib>`. Only then re-run with `arend -r --serialize <Module>`. Mention the archive path to the user so they can pick it up for serialization debugging.

---

## The metas, one paragraph each

### `rewrite p t` — rewrite in the goal

`p : a = b`. Replace `a` by `b` in the *expected* type, then `t` proves the rewritten type. Sugar for `transportInv (\lam x => goal[a := x]) p t`.

```arend
\func ex (n m : Nat) (p : n = m) (q : m = 0) : n = 0 => rewrite p q
\lemma +-assoc (a b c : Nat) : a + b + c = a + (b + c) \elim c
  | 0 => idp
  | suc c => rewrite (+-assoc a b c) idp
```

Variants:
- `rewriteI p t` ≡ `rewrite (inv p) t` — when the equation is wrong-way-round.
- `rewrite {n_1, n_2, ...} p t` — only the listed occurrences (1-indexed, left-to-right after each substitution). Use when the LHS appears multiple times and rewriting all chases its tail.
- `rewrite (p_1, p_2, ..., p_n) t` — tuple form, same as nested `rewrite`s. **Combine with `equation.cRing` as a workhorse**: `rewrite (lemma1, lemma2) equation.cRing` sets up a goal for the CRing solver after applying multiple algebraic identities. In-tree examples of the tuple-then-solver shape: `Algebra/Module.ard:402` (`rewrite (i.func-+, j.func-+) equation`), `Algebra/Field/Algebraic.ard:81` (`rewrite (f.func-+, f.func-negative, f.func-*) equation`), `Algebra/Linear/Matrix.ard:423`.
- `rewriteF p t` — forces `rewrite` to inspect the type of `t` rather than the expected type. Use when the LHS is missing from the goal but present in `t`'s type.
- `rewriteEq p t` — `rewrite` for subexpressions equal up to monoid/category laws (associativity, identity). Useful when `rewrite` can't see a syntactic match because of a different parenthesization.

### `run { f_1, f_2, ..., f_n, t }` — chain unary functions

Expands to `f_1 (f_2 (... (f_n t)))`. Last entry is the seed. Mix freely: `rewrite p`, `pmap suc`, `(p *>)`, any unary path-function.

```arend
\func ex {A : \Type} (a b c d : A) (p : a = b) (q : b = c) (r : c = d) : a = d
  => run { rewrite p, rewrite q, rewrite r, idp }
```

Debugging tip: replace any entry (or the seed) with `{?}` to surface the goal type at that point. The error message becomes a window into the intermediate state — useful when a chain fails halfway through and you need to know what the goal looked like before it broke.

If a middle entry fails with a confusing unresolved-goal error, prefix it with `later $` (see common failure mode #9).

**Inside the per-x body, Arend infers most implicits.** Once you're inside an enclosing `BigSum-ext`/`FinSum-ext`'s per-index lambda, the goal type is `arr1 x = arr2 x` (both sides constrained). The intermediate types in any `*>` chain inside that body are then unifiable by Arend, so the chain steps can drop their `{...}` annotations:

```arend
-- outside per-n body: every implicit explicit
A.FinSum-rdistr {FSeriesRing.PairsFinSet (n : Nat)}
                {pow c n k}
                {\lam s => a s.1 * b s.2}
*> AbMonoid.FinSum-ext {A} {FSeriesRing.PairsFinSet (n : Nat)}
                       {long-arr-1} {long-arr-2}
                       (\lam s => …)

-- inside the per-n body (arr1 n = arr2 n constrains both ends): just the punch lines
A.FinSum-rdistr *> AbMonoid.FinSum-ext (\lam s => …)
```

This is the dominant source of compression in `LHS-eq-M`-style proofs: ~3× reduction comes not from `run`/`rewrite` itself but from this argument-elision inside the per-x bodies.

**After the first anchored step in an outer `*>` chain, later entries also drop their implicits.** Once `BigSum-ext {A} {suc k} {explicit arr1} {explicit arr2} body` pins the chain to `BigSum arr2`, subsequent entries `*> inv (A.FinSum=BigSum {suc k})`, `*> A.FinSum-double-dep (\lam n => …)`, etc. can be argument-stripped — Arend recovers them from the previous step's output type. Keep explicit only what Arend genuinely can't recover:

- FinSet labels for SigmaFin variants (`{LHS-tri-fin k}` vs. `{LHS-rect-fin k}` — Arend can't decide between equivalent SigmaFin's).
- The dependent inner family of `FinSum-double-dep` (`(\lam n => PairsFinSet (n : Nat))`) — Arend can't guess that.
- The very first chain step's arrays (anchor for everything downstream).

If you find yourself writing arrays at the 3rd or 4th `*>` entry, suspect they can be elided. Test by deleting one at a time.

**`\lam x,` opens a binder inside `run`.** When a `run` entry expects a `\Pi (x : T) -> ...` argument (e.g. the per-index proof of `BigSum-ext`, or any partially-applied combinator), use `\lam x,` and the *rest of the `run` block* becomes that lambda's body. So

```arend
run { ..., BigSum-ext {l} {l'}, \lam i, rewrite p, rewrite q, idp }
```

is equivalent to

```arend
... *> BigSum-ext {l} {l'} (\lam i => rewrite p (rewrite q idp))
```

This is the workhorse pattern for finite-sum proofs.

**Slot-notation composers for inequality chains.** `` \`<∘ <strict-bound> `` is `\lam h => h <∘ <strict-bound>` — a partially-applied composer. Used in a `run` block, it collapses `<∘`-chained inequalities:

```arend
real_<_U.1 (run {
  rewrite y-eq,             -- align goal
  `<∘ (real_<_U.2 q.3),     -- compose with strict bound
  linarith                   -- close residual
})
```

The general shape is: align the goal with `rewrite`, compose forward through each known bound with `` `<∘ `` / `` `<=∘ ``, then let `linarith` close whatever numeric slack is left. Reach for it whenever a proof would otherwise name five intermediate inequalities.

### `f in x` — apply `f` to `x`, dropping the expected type

`\let r => f x \in r`, with `\let` typechecked without goal-type info. Mostly invisible — except for `rewrite`, which depends on the expected type. Use `f in x` when the meta should look at `x`'s type instead.

```arend
\func ex (x y : Nat) (p : zero = x) (q : zero = y) : x = y
  => rewrite p in q   -- rewrites zero -> x in q's type
```

`in` has loose precedence (priority 1, right-assoc). To apply `f in x` to more args: parenthesize. `(simp_coe in t) b` differs from `simp_coe in t b`.

### `f at h` — modify a local binding

Shadows `h` with `f h` for the rest of the expression. Three equivalent forms:

```arend
(rewrite p at q) (\case q \with {})
rewrite p at q $ \case q \with {}
run { rewrite p at q, \case q \with {} }
```

Tuple form: `(f1, f2) at h` ≡ `f1 (f2 h)` (each step applied in order, type stripped). Works on parameters, `\let`-, and `\have`-bindings.

### `cases (e_1, ..., e_n) \with { ... }` — multi-scrutinee `\case`

Avoids `\case e_1, e_2 \with` boilerplate. With `arg addPath`, also adds an `idp`-equality between the scrutinee and the matched pattern, and rewrites the dependent return type automatically:

```arend
\func baz {A : \Type} (B : Bool -> \Type) (p : A -> Bool) (a : A)
          (pt : B true) (pf : B false) : B (p a)
  => cases (p a arg addPath) \with {
    | true,  q => pt
    | false, q => pf
  }
```

### `mcases {f}` — match the `\case` already in the goal

When the expected type contains `f n` and `f` is itself defined by `\case`, `mcases {f}` builds a `\case` that mirrors `f`'s clauses:

```arend
\func test-f (n : Nat) : f n = 5 => mcases {f} \with { | 0 => idp | suc _ => idp }
```

Variants:
- `mcases \with { ... }` — finds `\case`-expressions (no `f` to point at).
- `mcases {(f, k)}` — only the k-th occurrence of `f` (1-indexed).
- `mcases {f, g}` — multiple definitions at once.

### `simplify` — algebraic normalizer

Strips `ide`, double inverses, `zro`-like absorbing elements. Inspects the in-scope `Monoid`/`Group`/`Ring`/`Semiring` instance.

```arend
\func ex {M : Monoid} (x : M) : x = x * ide => simplify
\func neg {R : Ring} (x : R) : negative (negative x) = x => simplify
```

`simplify in h` rewrites a hypothesis. **Does not** know commutativity or distribution. **Can fail** on goals that *should* normalize — when this happens, fall back to `equation` or manual `rewrite`.

**Try `simplify` before `equation.cRing` for concrete-record component identities.** Goals like `0 * 0 + 0 * 0 = 0` over a concrete `Complex`, or `0 + x = x` on a named instance, dispatch with `simplify` more cleanly than with `equation` (which over-fires the ring solver and reports `Ring solver failed` if `*-comm` is needed but absent). The 2026-05-15 arend-lib refactor replaced ~10 calls of `=> equation` with `=> simplify` for `cabs2_zro`, `cabs2_ide`, `cabs2_negative`, `cabs2_i`, and similar — `simplify` is both shorter and a more accurate description of what the proof is doing.

### `equation a_1 ... a_n` — chained equality, steps inferred

Proves `a_0 = a_{n+1}` via the intermediates. Each step proof is searched in the local context; if not found, supply it as an implicit.

```arend
\func ex {M : Monoid} (x y z : M) (p : x = y) (q : y = z) : x = z
  => equation x y z
```

Use `equation` when step proofs are uninteresting. Use `==<` / `>==` / `qed` when each step deserves a name in the proof body.

**`equation` defaults to the non-commutative ring solver.** If your identity needs `*`-commutativity, plain `equation` will fail with `Ring solver failed`, and the printout shows the two sides differing only by argument order (e.g. `v0*v1*v2` vs `v2*v0*v1`). Use the variant matching your structure:

- `equation.cRing` — commutative rings (`Rat`, `Real`, etc.).
- `equation.cMonoid` — commutative monoids (no addition involved).
- `equation.monoid` — non-commutative monoids.

Existing arend-lib calls (`Algebra/Ring/Localization.ard`, `Algebra/Linear/Matrix.ard`) already pick the correct variant — but `equation` alone is the wrong default for ordered-field arithmetic and the failure message doesn't directly suggest the fix.

### `cong` — congruence closure

Solves `f x_1 ... x_n = f y_1 ... y_n` given the equalities `x_i = y_i` somewhere in scope. Multi-argument generalization of `pmap`. Note: target type lives in `\Type`, so use `\func` (not `\lemma`).

```arend
\func ex {A B : \Type} (f : A -> A -> B) (x x' y y' : A) (p : x = x') (q : y = y')
       : f x y = f x' y' => cong
```

### `ext` — extensionality (polymorphic)

| Goal shape | Subgoal |
|---|---|
| `f = g` for `f, g : \Pi (x : A) -> B x` | `\Pi (x : A) -> f x = g x` |
| `t = s` for `\Sigma A B` (or record) | `\Sigma (h_1 : t.1 = s.1) ... ` (with `coe` corrections for dependents) |
| `A = B` for `A, B : \Prop` | `\Sigma (A -> B) (B -> A)` |
| `A = B` for `A, B : \Type` | `Equiv {A} {B}` |
| `x = y` with target in `\Prop` | *no subgoal* — meta closes it |

```arend
\func fun-eq (f g : Nat -> Nat) (h : \Pi (n : Nat) -> f n = g n) : f = g => ext h
\func sig-eq (p p' : \Sigma Nat Nat) (h1 : p.1 = p'.1) (h2 : p.2 = p'.2) : p = p'
  => ext (h1, h2)
\lemma prop-eq (A B : \Prop) (f : A -> B) (g : B -> A) : A = B => ext (f, g)
```

Record copattern form: `ext R { | f_1 => p_1 | f_2 => p_2 | ... }`.

### `linarith` — Fourier–Motzkin for linear arithmetic

Dispatches on the in-scope `LinearlyOrderedSemiring` / `OrderedRing` instance. Reads hypotheses from the context automatically — you don't pass them explicitly when they're already named in scope.

```arend
\lemma nat-1 {a b : Nat} : 0 <= a Nat.+ b => linarith
\lemma combine {a b c : Nat} (p : a <= b) (q : b Nat.+ c <= a) : c = 0 => linarith
\lemma generic {R : LinearlyOrderedSemiring} {a b c : R}
               (p : a <= b) (q : b + c <= a) : c <= 0 => linarith
```

Out of scope: products of variables, exponentiation, anything nonlinear.

**Over `Real`, `\import Arith.Real.Field` is enough** (since commit `ab3dc1dba`). `linarith` resolves the `LinearlyOrderedSemiring Real` instance through the `solveEquationsFor` inference push, with or without `\open RealField`, and with or without multiple `linarith` calls in the same file. If you still see `Cannot infer an instance of class 'LinearlyOrderedSemiring' with classifying expression: Real`, check the import is actually present.

**`equation.cRing` vs `linarith` — pick the right tool.** `linarith` is for *linear* (in)equalities; `equation.cRing` is the commutative-ring identity solver. A goal like `(A - B) - (A + B) = negative (2 * S * S')` is linear in atoms `A`, `B`, `S*S'`, but `linarith` can't normalize `natCoef 2` against literal `2`, and can't push `negative (...)` through the right slot — `equation.cRing` handles it instantly. The diagnostic to switch is: if `linarith` complains "Cannot solve the equation" and prints two algebraically-equivalent sides, you're in the ring-identity case, not the linear-(in)equality case. Reach for `equation.cRing`.

`linarith` does not need redundant context hints. `linarith delta_min>0 : T` and `linarith : T` close the same goal when `delta_min>0` is named in scope. The explicit-hint form is for when the hypothesis isn't visible (deep inside a `\let` that shadows it). Strip the hint on the final pass.

### `simp_coe` — push `coe`/`transport` through structure

For accumulated `coe`/`transport` over Π, Σ, or records. Most useful in HIT-heavy Part II proofs. Niche; reach for it only when the goal is `coe`-ridden.

### `unfold (f, g, ...) e` — unfold definitions in the expected type

Before checking `e`, unfold the listed definitions in the goal. Tuple form runs through them left-to-right. Single-name form `unfold f e` works without parens.

**Unfold minimally.** A `\func f` already reduces definitionally during elaboration, so `unfold f` is only needed when Arend's elaborator doesn't see through `f` on its own — usually when the goal involves *projections* from `f`'s result, or when `f`'s body matters for unifying a `rewrite`/`pmap` match. Example: `unfold (LHS-y, LHS-f)` got trimmed to `unfold LHS-f` because the goal mentioned `(LHS-f x).1` and `(LHS-f x).2`, which Arend couldn't reduce through; the surrounding `LHS-y` reduced on its own. Try dropping each `unfold` argument and see what fails — leftover entries earn their keep.

```arend
\func sq (n : Nat) => n * n
\lemma ex (n : Nat) : sq n = n * n => unfold sq idp
```

`unfold_let e` does the same for `\let`-bindings in scope. Use when the goal references a `\let`-bound name and you want it expanded before further work.

### `assumption` — fill from the local context

Looks up the goal type among the in-scope hypotheses, latest binding first, and uses the first match. `assumption {n}` skips the first `n` matches (1-indexed). Useful in `\case` arms where the right witness is already in scope under some name you don't want to hunt for.

### `using (e_1, ..., e_n) e` — scope control around a sub-term

- `using (e_1, ..., e_n) body` — typechecks `body` with `e_1, ..., e_n` added to the local context (as anonymous `\have`s). Lets you make a derived fact visible only inside one expression.
- `usingOnly (e_1, ..., e_n) body` — like `using`, but *replaces* the visible context with just those entries. Forces metas like `assumption`, `linarith`, `cong` to look only at the listed facts. Helpful when too many irrelevant hypotheses are confusing a search-based meta.
- `hiding (e_1, ..., e_n) body` — typechecks `body` with the listed names removed from scope.

### `$` / `#` — application operators that respect implicit args

From `Function.Meta`. Behave like Haskell's `$` but participate in implicit-argument inference and section syntax.

- `f $ x` — right-associative, low priority. `f $ g $ x` ≡ `f (g x)`. Avoids parens around long argument expressions.
- `x # f` — left-associative dual. `x # f # g` ≡ `g (f x)`. Good for "pipe" reading.
- `__ $ a` — section with the implicit-lambda placeholder `__`. `__ $ a` ≡ `\lam f => f a`.

### `later meta args` — defer a meta invocation

From `Meta`. Postpones the elaboration of `meta args` until the surrounding type-inference has made more progress. Most often used as `later $ rewrite p`, `later $ simplify`, etc., inside a `run` block where the intermediate expected type isn't yet pinned down. If a meta in the middle of a chain fails with "cannot infer expected type" or similar, `later` is the first thing to try — see common failure mode #9.

### `repeat f x` — iterate a meta to fixpoint

Applies the unary meta `f` to `x` until the result stops changing. `repeat {n} f x` caps at `n` iterations. Mostly used as `repeat (rewrite p) t` or `repeat unfold_let t`.

### `defaultImpl C F E` — invoke a class's default field implementation

Given class `C` with default implementation `F`, plus the explicit arguments `E` it expects, returns the default value. Use when an instance overrides a field but you want to refer to (or fall back to) the class-level default.

---

## Decision table

| If the goal looks like... | Reach for |
|---|---|
| `f x_1 ... x_n = f y_1 ... y_n` with each `x_i = y_i` in scope | `cong` |
| Linear (in)equality between sums/products with constants | `linarith` (over `Real`: just `\import Arith.Real.Field`) |
| Equation between monoid/ring expressions, no commutativity needed | `simplify` |
| Ring/field identity over `Rat`/`Real` or any commutative ring (needs `*`-commutativity) | `equation.cRing` — **plain `equation` will fail** |
| Symbolic identity in atoms with `negative`/`natCoef` (e.g. `(A-B) - (A+B) = -2B`) | `equation.cRing` — **`linarith` chokes on `natCoef` vs literal `2`** |
| Identity over a commutative monoid (no addition) | `equation.cMonoid` |
| Identity that bridges `finv` / `*-rat` / `fromRat` with CRing rearrangement | `rewrite (finv_*, inv *-rat) equation.cRing` |
| Equation that needs a *named* sequence of intermediate steps | `equation a_1 ... a_n`, or `==<`/`>==`/`qed` |
| Need to substitute one term in the goal | `rewrite p t` |
| Need to substitute, but the equation is reversed | `rewriteI p t` |
| Multiple instances of LHS, want only one | `rewrite {n} p t` |
| Several rewrites, all in goal | `rewrite (p_1, ..., p_n) idp` |
| Mixed chain (rewrite + pmap + sections) | `run { ..., idp }` |
| Inequality chain (`<` / `<=`) | `run { rewrite <align>, \`<∘ <bound>, linarith }` |
| Need to reshape a hypothesis before using it | `rewrite p at h $ body` |
| Need to feed a hypothesis to a meta when there's no goal type | `f in h` |
| `f = g` (functions) or tuple/record equality | `ext h` (with appropriate subgoal) |
| Prop-irrelevance bridge `F p = F p'` from `p.1 = p'.1` | `\elim p, p', h \| ..., ..., idp => pmap2 ... prop-pi prop-pi` |
| Goal contains a `\case` of a function I didn't write | `mcases {f}` |
| Need to remember the scrutinee's value while pattern matching | `cases (e arg addPath)` |
| `coe`/`transport` chains visible in the goal | `simp_coe` |
| Goal mentions a definition I want unfolded | `unfold f body` |
| Right witness is already in the context, just need to plug it in | `assumption` |
| Search-based meta confused by too many hypotheses | `usingOnly (h_1, ..., h_n) body` |
| Want one local derived fact visible inside one expression | `using (h) body` |
| Apply a meta until it stops doing anything | `repeat f x` |
| Long pipeline of unary functions | `f $ g $ x` or `x # f # g` |
| Meta inside `run { ... }` fails with unresolved-goal error | Prefix it with `later $` |
| Want to inspect intermediate goal in a `run` chain | Replace an entry with `{?}` and read the error |

---

## When a meta fails — quick triage

1. Read the actual error. `Cannot resolve reference 'X'` → missing import. `The type of a lemma must be a proposition` → `\lemma` → `\func`. `Type mismatch: Expected type: a class. Actual type: \Set0` → using `Nat.<=` or similar; switch to bare operators with `Arith.Nat`.

2. If `simplify` says `Meta 'simplify' failed`: the meta tried to normalize and the two sides didn't reconcile. Don't keep retrying — switch strategy. Try `equation` (with intermediates), or do one `rewrite` to align sides then `simplify`, or just write `pmap`/`*>` by hand.

3. If `cong` fails on something that looks obvious: check that the relevant `x_i = y_i` is actually visible in the local context (not buried inside a `\Sigma` or `\let`-bound under a different name).

4. If `linarith` fails: first check whether the goal is genuinely linear (atoms multiplied by constants, summed, compared) — if it's a polynomial identity like `(A-B) - (A+B) = -2B`, switch to `equation.cRing`. If the goal IS linear and the error is `Cannot infer an instance of class 'LinearlyOrderedSemiring'`, the instance isn't in scope — for `Real`, add `\import Arith.Real.Field`; for `Rat`, `\import Arith.Rat`; for `Nat`, `\import Arith.Nat`. Since commit `ab3dc1dba` (`solveEquationsFor` inference push), `\open RealField` / `\open RatField` is no longer required. If the import alone doesn't fix it, check that no local tutorial-style class is shadowing the real one.

5. If `rewrite` complains it can't find the LHS: the LHS in `p` may already be reduced away in the goal (definitional unfolding). Try `rewrite p in t` to operate on `t`'s type, or `rewrite` a hypothesis instead.

6. If `ext` produces a weird subgoal with `coe` in it: the equality is between dependent Σ/record values and the meta inserted a transport. Either prove the corrected goal directly, or follow up with `simp_coe`.

7. If `equation` fails with `Ring solver failed` and the printout's two sides differ only by argument order (`v0*v1*v2` vs `v2*v0*v1`): you need commutativity. Switch to `equation.cRing` (or `equation.cMonoid` if no addition). The plain `equation` solver is non-commutative; commutative is opt-in via the qualified variant.

---

## Looking things up — cheap to expensive

When you're mid-proof and need to find a lemma, a class field, a norm law — the temptation is to `grep -r` the sources. **Resist.** The cost ordering is:

1. **`-ss <name>` for name lookup** — fast, served from a per-library on-disk index. Prints each match's location, qualified name, kind, and a **one-line signature**, so it often answers the question outright without opening anything. Start here whenever you can guess any part of the name.
2. **`-ss <Module>.<name>` when you know the owner** — a dotted pattern matches parts of the qualified name, so `-ss Monoid.*-comm` narrows to one module's members instead of the whole library.
3. **`-ps "<shape>"` for shape lookup** — slow (~20s) but the right tool when you know the type and not the name. Negative results are equally valuable.
4. **`-ch <class>` / `-fu <name>` for structure** — which classes extend what and which instances exist (`-ch`), or how a definition is actually used at its call sites (`-fu`).
5. **Sources (`arend-lib/src/<module>.ard`) last** — only when you need the proof body, a precise reduction, or to confirm a default implementation. Read the signature line and skip the body. Don't read sources to *find* something — only to *understand* it.

**Default discipline: when about to write `\lemma foo : <shape> => {?}`, run `-ps "<shape>"` first.** When about to grep sources for "where does `norm_+` live?", run `-ss norm_+` instead — the one-line signature in the result is usually the whole answer.

### `-ss` dialect prefixes — concrete examples

The default is **case-insensitive substring on short names**, *not* regex. Reaching for `.*` or `.+` means you guessed wrong:

| Want | Wrong (guessed regex) | Right |
|---|---|---|
| Names containing `BigSum` and `<=` | `-ss "BigSum.*<="` | `-ss BigSum contains=<=` |
| Names containing `midSum` and `rdistr` | `-ss "midSum.*rdistr"` | `-ss midSum contains=rdistr` |
| Names matching `norm_*_<=` literally | `-ss "norm_*_<="` (works by accident) | `-ss "lit:norm_*_<="` (intent-clear; `eq:` was removed) |
| Either of two name patterns | `-ss "A\|B"` (shell escape, no-op) | `-ss A -ss B` (multiple flags = OR) |
| Glob-style wildcard | `-ss "Big*<="` (literal) | `-ss "glob:Big*<="` |
| Genuine regex | `-ss "^abs.*_+"` (literal) | `-ss "re:^abs.*_\+"` |

**Rule: if you typed `.*`, `.+`, or `\|` into a `-ss` query, back up and pick a dialect prefix or `contains=`.** Multiple `-ss` flags OR; `contains=` adds AND-substring filters; `limit=N` caps results natively (no need to pipe to `head`).

`-ss` searches **short names only** — `Module.Foo` queries match `Foo`. To find a method on a class, query the bare method name and read the qualified result.

---

## When to reach for `run { {?} }`

`run { m1, m2, ..., t }` chains metas left-to-right around a seed term `t`. Useful for *step-by-step refinement* of a goal where each step is a known meta application.

**Reach for it when:**

- The proof is dominated by **rewrite chains** (`rewrite p`, `rewrite q`, `simplify`) and you want to discover them one at a time. Wrap the goal in `run { {?} }`, then iteratively replace the inner `{?}` with `rewrite <something>, {?}` and re-typecheck to see what the new goal looks like.
- You're doing **goal manipulation** (transports, rewrites, simplifications) rather than building a value out of named pieces.
- The proof is a **`<∘`-chained inequality** — `run { rewrite <align>, \`<∘ <strict-bound>, linarith }` collapses what would otherwise be 5+ named intermediates.
- You want a more *imperative-style* proof — "first do this, then this, then close with `idp`."

**Don't reach for it when:**

- The proof's bottleneck is *mathematical content*, not *tactic plumbing*. If the inductive-step bound itself fails (counterexample), no `run` chain will help — the lemma needs a different argument.
- You're assembling a value out of named helpers (`\have | h1 => ... | h2 => ... \in combine h1 h2`). The `\have` chain is clearer there.

A practical hybrid: use `\case \elim n` to set up induction, prove the trivial base, then for the step replace `{?}` with `run { {?} }` and refine inside.

---

## Style — when to prefer which surface form

- **Pure rewrites on the goal**: `rewrite (p, q, r) idp` is shorter than a `run`-block. Use it.
- **Multiple rewrites + an algebraic close**: `rewrite (p, q) equation.cRing`. One line.
- **Mixed chains**: use `run`. The vertical layout reads better than nested calls.
- **Single-step manipulation of a hypothesis**: `f at h $ body`. Two-step+: `run { f1 at h, f2 at h, body }`.
- **Equation with three or more steps and you want each to be named**: use `==<`/`>==`/`qed`, not `equation`. The named form is what the reader will navigate.
- **Equation where every step is `simplify` or trivial**: use `equation` with no explicit steps if possible.
- **Prop-irrelevance bridge** (`F p = F p'` from a single component path): use multi-`\elim` with `idp` in the same clause list, not a nested helper.

---

## Style — compaction

Apply these while writing, not only on a post-pass:

- **Delete** idp-bodied lemmas and `pmap f idp` steps — they carry no content.
- **Inline** single-use helpers and single-use `\have` clauses.
- **Replace** hand-stepped `\have` chains with `simplify` / `equation.cRing` / `linarith` where one of them closes the goal outright.
- **Flatten** with slot notation (`` `<∘ b ``, `__ + 1`) and `$` instead of nested parens.
- **Migrate** helpers used by exactly one parent into that parent's `\where`.
- **Alias** hot long names file-locally with `\open M \using (Long \as short)` — e.g. `\open Complex \using (iunit \as i)` (`Arith/Complex.ard:19`).
- **Hoist** repeated explicit embeddings to `\use \coerce` — see **arend-quirks** §10.
- **Prop-irrelevance bridges**: multi-`\elim` with the path in the same clause list.
- **Tuple-then-solver**: `rewrite (p, q) equation.cRing` rather than sequential rewrites.
- **Doc comments**: prefer `-- | …` one-liners; `{- | … -}` nests, so LaTeX braces like `x^{-1}` can swallow the rest of the file — see **arend-quirks** §19.

(These were previously kept in a separate `arend-refactor` skill. That skill's `SKILL.md` had been overwritten with a duplicate of `arend-error-extraneous-input`, so it was deleted on 2026-07-28; this list is the surviving summary.)

---

## Log friction

Anything that felt unreasonable about the **tooling** — surprising typechecker errors, instance inference that should "just work", misleading diagnostics, slow CLI paths — goes to `arend-issues.md` at the repo root (`/home/sergey/Documents/Arend/arend-issues.md`) with a one-liner on where it bit you. Future you needs the trail.

**State of that file: deleted 2026-05-21 (`f0c5f9dbf`, "Remove issues/blueprint"), recreated 2026-08-03** at the user's request to hold CLI/typechecker feature requests derived from the two "Refactor AI slop" reviews. Its old numbered entries are gone, so never cite "`arend-issues.md` #N" — append new entries instead.

**Missing library statements do not belong here.** A gap in arend-lib is a contribution opportunity, not a complaint to file: add the helper directly to the appropriate module. `arend-issues.md` is for things the agent *cannot* fix by editing arend-lib — tooling and language-level friction.

---

## Anti-patterns

- **Iterating proof attempts without `--serialize`.** Serialization is opt-in, so without the flag nothing is cached and every cycle re-typechecks arend-lib from source — ~2 min per attempt instead of seconds. Turning it off is for diagnosing a suspected cache bug, not a default.
- **Attacking a 200-line proof as one expression.** Decompose first, attempt second.
- **Filtering typecheck output with `grep "GOAL"`.** Narrow the *scope* instead of the *output* — `arend --serialize <Module>:<def>` checks only the def you care about, and grep would cut off the `Expected type:` / `Context:` lines that make a `[GOAL]` block worth reading.
- **Working around a missing 3-line helper inline.** Add it to the right module; everyone benefits. (And don't log it as an "issue" — just add it.)
- **Spending an hour on `transport (\lam x => x.U _) p t` when `rewrite p` works.** Recognize the "Expected type: a class" symptom early.
- **Treating a typechecking proof as "done" without re-reading it.** Same caveat as the formalize skill — typechecking proves consistency, not faithfulness. Verify the proof closes the *intended* mathematical goal.
- **Stating a helper lemma before running `-ps "<its shape>"`.** Cheaper to learn the helper exists (or doesn't) than to write a redundant skeleton.
- **Sprinkling `Module.op` qualifiers when the ambient instance suffices.** Bare operators resolve through type inference for primitive operations; only qualify named lemmas that aren't in the open set.
- **Grepping `arend-lib/src/` to find a lemma.** `-ss` prints a one-line signature per match at a fraction of the tokens, and `-ps` finds things by shape when you can't guess the name at all. Sources are for reading proof bodies, not for navigation.
- **Typing `.*` or `\|` into a `-ss` query.** That's regex muscle memory on a substring-by-default tool. Use `contains=` for AND, multiple `-ss` flags for OR, or `re:`/`glob:` prefixes when you genuinely need a wildcard.
- **Naming every intermediate value in a proof you're writing for the first time.** The compaction idioms under *Style — compaction* above (multi-`\elim` with paths, `rewrite (tuple) equation.cRing`, `run` with slot-notation composers) are the target style *from the first attempt*, not just post-pass. Try the compact form first — the typechecker tells you fast if you need to expand.
