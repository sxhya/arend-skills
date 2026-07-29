---
name: arend-error-type-mismatch
description: Diagnose misleading Arend `Type mismatch: Expected X, Actual Y` errors where the printed types look wrong, unrelated, or contain metavariables. Use whenever an .ard typecheck error of this shape doesn't make sense — the literal types are downstream effects of an upstream name-resolution or implicit-inference failure. Catalogue of known causes with minimal repros and fixes.
user-invocable: true
---

# Arend typechecker error — `Type mismatch: Expected X, Actual Y`

Arend's `Type mismatch` error reports the types the elaborator computed *after* elaboration finished. When the printed `Expected` / `Actual` don't make sense for the code you wrote — wildly unrelated types, metavariables on one side, or a value where you expected an equation — the real failure happened earlier:

- a name resolved to the **wrong** definition (Arend rarely says "unresolved"; it silently picks something plausible);
- an implicit argument (especially a class instance) **failed to infer** and got defaulted to a bogus value;
- a coercion or `\peval` reduction was inserted/skipped because of one of the above.

Workflow when this fires:

1. Look at the `Expected` type. If it mentions `I` (the interval), `Path`, `?a = ?a'`, or any other shape your source code didn't ask for, the elaborator inserted/inferred something — find what.
2. Look at the `Actual` type. If it's the type of *the thing you intended* but in the wrong slot, suspect resolution / instance inference rather than your math being wrong.
3. Check that every name you wrote actually points at what you think (qualify it explicitly).
4. Check that every implicit instance argument can be inferred at the call site (pass it explicitly to test).
5. Once root-caused, append the case to **Known causes** below.

---

## Known causes

### Mis-resolved `\protected` lemma from a `\where` block (2026-05-08)

**Repro:** Inside `BigSum-rdistr-bounded` in `Analysis/CauchyProduct.ard` (that lemma no longer exists under this name; the rule below is unchanged), reaching for `zro_*-left` — which is `\protected` inside `ExUpperRealSemigroup.\where` and takes an `IsBounded` argument (`Arith/Real/UpperReal.ard:412`), while `Algebra/Semiring.ard:17` defines an unrelated field of the same short name:

    ... zro_*-left ...    -- unqualified

**Error:**

    Type mismatch:
      Expected: I
      Actual:   TruncP (\Sigma (r : Rat) (z.U r))

— an `IsBounded` argument being fed to a path-type slot. The `I` (interval) tipped off that Arend had resolved `zro_*-left` to *some other* same-named lemma whose first argument was a path.

**Fix:** Qualify the name:

    ExUpperRealSemigroup.zro_*-left

**Why:** `\protected` definitions inside `Foo.\where` are not brought into scope by unqualified name from outside `Foo`. Arend does **not** say "no such reference" — it silently resolves to a different `zro_*-left` (or fails resolution in a way that defaults to the wrong slot) and reports the *downstream* type mismatch.

**Diagnostic:** When the Expected type contains `I` / a path shape and you didn't write a path-valued expression, suspect mis-resolution. Try fully qualifying every cited lemma as the first debugging step before chasing the types.

---

### `\peval f …` with un-inferrable implicit instance (2026-05-08)

**Repro:** Inside `fubini-inner-eq` in `Analysis/CauchyProduct.ard` (since renamed — the Ring analog `fubini-inner-conv` at `Analysis/CauchyProduct.ard:219` still carries a doc comment naming the original):

    \peval partialSum (\lam k => a k * partialSum b (suc n -' k)) n

where `partialSum` (`\sfunc partialSum {A : AddMonoid} (S : Series A) (n : Nat) : A`, `Analysis/Series.ard:51`) has an implicit `{A : AddMonoid}` parameter that the call site can't infer.

**Error:**

    Type mismatch:
      Expected: ?a = ?a'
      Actual:   ExUpperReal

The Expected is a path with two metavariables — i.e. Arend wanted `\peval` to return a path proof but instead got the body value `ExUpperReal`.

**Fix:** Make the instance explicit:

    \peval partialSum {ExUpperRealAbMonoid} X n

**Why:** When `\peval f x y` can't infer `f`'s implicit instance, it apparently falls back to producing the *body* of `f` itself (a value of `f`'s result type) rather than the equation `f x y = body`. The downstream consumer expected a path and got a value — hence Expected `?a = ?a'` vs. Actual `ExUpperReal`.

**Diagnostic:** When the Expected side of a `Type mismatch` is `?a = ?a'` (or any `Path …` with metavariables) and the Actual side is the result type of the function you `\peval`'d, suspect that `\peval`'s implicit instance didn't infer. Pass the instance explicitly. `\peval` is not exempt from ordinary instance-inference hygiene.

---

---

### `pmap (\lam c => f (i * c))` — lambda binder ambiguous under a coercion (2026-07-28)

**Repro:** rewriting under `exp (i * _)` where `i : Complex` and the intended binder is a `Real`, relying on `Complex.fromReal`'s `\use \coerce`:

    pmap (\lam c => exp (i * c)) (someRealEquation)

**Error:** a `Type mismatch` between two `exp {ComplexBanach} (\new Complex … )` terms whose components differ by stray `re {…}` / `im {…}` projections:

    Expected type: exp (\new Complex (zro * (natCoef 0 * a) + negative (ide * zro)) …) = ?a'
      Actual type: exp (\new Complex (zro * re {natCoef 0 * a} + negative (ide * im {natCoef 0 * a})) …) = …

**Fix:** either annotate the binder — `pmap (\lam (c : Real) => exp (i * c)) …` — or, better, name the map once and `pmap` that:

    \func eit (a : Real) : Complex => exp (i * a)
    ...
    pmap eit someRealEquation

**Why:** inside the lambda, nothing forces `c`'s type. Arend can read `i * c` either as `Complex * Complex` with `c` already complex, or as `Complex * fromReal c` with `c : Real`. It picks per-occurrence, so one side of the equation elaborates with the coercion inserted and the other without; the mismatch then surfaces as `re {…}` / `im {…}` projections applied to what should have been a plain real.

**Diagnostic:** a `Type mismatch` between two structurally identical `\new Complex …` terms where one side has `re {x}` / `im {x}` and the other has plain `x`. That asymmetry means a coercion fired on one side only — look for an unannotated lambda binder, not for a wrong lemma.

---

## How to extend this skill

When you fix a fresh `Type mismatch` whose root cause isn't in the catalogue above:

- Add a new `### <short cause tag> — <date>` subsection under **Known causes**.
- Include: minimal repro, the misleading Expected/Actual, the fix, the why, and a one-line **Diagnostic** describing the shape of the message that should trigger this case next time.
- Keep entries dated so stale ones can be culled when Arend's elaborator changes.
