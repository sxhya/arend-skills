---
name: arend-quirks
description: Survival notes for actually formalizing things in Arend. Use when writing .ard files, reviewing Arend proofs, explaining why Arend syntax/semantics differ from Coq/Agda/Lean, or debugging an Arend typecheck failure. Written as "things that surprised me after reading the tutorial and language reference".
user-invocable: true
---

# Arend quirks — what the docs say vs. what I'd naively expect

This is a personal cheat-sheet of places where Arend deviates from the Coq/Agda/Lean-shaped picture I carry around by default. Not a reference manual — the Arend language reference is. These are the paper-cuts I want to remember before writing a proof.

Sources referenced below live in the **website** repo, `/home/sergey/Documents/site/` — not in the Arend or arend-lib tree: `src/documentation/tutorial/PartI/*.md`, `src/documentation/tutorial/PartII/*.md`, `src/documentation/language-reference/*`, and `src/documentation/standard-tactics/*.md`. (Other checkouts of the same docs exist at `/home/sergey/Documents/Site/documentation/` and `/home/sergey/Documents/arend-lang.github.io/documentation/`; the `src/documentation/…` paths quoted below match the `site/` layout.)

Library citations of the form `Module/File.ard:NNN` are relative to `/home/sergey/Documents/Arend/arend-lib/src/` and were re-verified against branch `staging` on 2026-07-28.

Companion skills:
- **arend-formalize** — procedural workflow for going from an informal math statement to a typechecking `.ard` file (which CLI tools to use, in what order).
- **arend-prove** — catalog of metas (`rewrite`, `ext`, `simplify`, `linarith`, …) for closing equality and algebra goals.

---

## 1. Equality is not an inductive family

What I'd expect: `Id A a b` / `Eq A a b` / `a = b` is an inductive with one constructor `refl`, eliminator `J`.

What Arend actually has (`language-reference/prelude.md`, `tutorial/PartI/idtype.md`):

- `Path (A : I -> \Type) (a : A left) (a' : A right)` is the primitive. Terms are functions `f : \Pi (i : I) -> A i` that **additionally** satisfy the definitional side-conditions `f left => a` and `f right => a'`. These boundary conditions are enforced by the typechecker when you write `path (\lam i => ...)` — a boundary mismatch gives a dedicated error.
- `a = b` is defined notation for `Path (\lam _ => A) a b`.
- Reflexivity is `idp := path (\lam _ => a)`. There is no `refl` constructor; `idp` is treated as a constructor for pattern matching only (it has no real body).
- `J` is not primitive; it's derived from `coe` (the interval eliminator). Practically you use `transport`, `pmap`, or pattern-match on `idp`.
- There is a definitional eta rule `path (\lam i => p @ i) == p`, but only for the built-in `@`.
- Implicit coercion between paths and `\Pi (i : I) -> A i` fires automatically (`path` / `@` inserted).

Consequence: when a goal contains `p @ left` or `p @ right`, it *reduces* to the endpoint. Use this aggressively — it's not propositional, it's definitional.

## 2. Pattern-matching on `idp` has a subtle restriction

What I'd expect: match `p : a = b` with `idp`, done.

What the docs spell out (`prelude.md`, `tutorial/PartI/equalityex.md`, `tutorial/PartI/case.md`):

- When you match `p : e = e'` with `idp`, at least **one** of `e`, `e'` must be a variable that does **not** occur in the other. "Matching on `idp`" is really matching on the `Σ`-pair `(a', p)` which has one inhabitant.
- The replaced variable's free variables must all be bound at each of its occurrences (substitutability condition).
- In `\case`, you must additionally bind the endpoint with `\elim` or put the equality arg in a position where the variable comes from the case scrutinee. A common trick is `\case expr \as x, idp : x = expr \with { ... }` to reconnect the matched value with the original expression.
- `J` via pattern matching:
  ```arend
  \func J {A : \Type} {a : A} (B : \Pi (a' : A) -> a = a' -> \Type) (b : B a idp)
          {a' : A} (p : a = a') : B a' p \elim p | idp => b
  ```

If you get "cannot match on idp here", the first thing to check is whether both sides of the equality are non-variables or share variables.

## 3. `\case` vs `\elim` vs `\with` is not just taste

What I'd expect: these are interchangeable styles.

What actually differs (`tutorial/PartI/case.md`, `tutorial/PartI/synndef.md`, `language-reference/definitions/functions.md`):

- `\func f x \elim x | ... => ... | ...` — `f n` **reduces** to the matched body. Normal forms look nice.
- `\func f x => \case x \with { | ... => ... }` — `f n` stays as a `\case`-expression in normal form. Worse UX downstream.
- Prefer top-level `\elim`/`\with` whenever possible; reach for `\case` only for scrutinizing *expressions* that aren't parameters.
- Termination checker **only tracks eliminations at the top level** (`\elim`, `\with`). A decreasing `\case` inside the body does not count — you'll get a termination error even when obvious. Factor out a helper with `\elim` in that case.
- In `\case`, use `\case \elim x \with { ... }` to re-substitute `x` in the return type per clause. Without `\elim`, `x` stays as `x` in the return type and you'll often need `\return ... \with { }` plus a `\as`-binder.
- `\case e \as b \return T[b] \with { ... }` is the standard dependent-case idiom.
- `\scase` is a strict variant (parallel to `\sfunc`).

**Reducing `\elim`-defined functions in lemmas: eliminate the dependent variable that the *constructor*'s recursive arg lives in.** When a function `f` is defined via multi-pattern `\elim k, x` over a data type with implicit indices, `f (C … rest)` (where `C` is a constructor and `rest` is its recursive sub-component) does NOT reduce in a downstream lemma unless the lemma `\elim`s **`rest`** (in addition to `k`).

Intended Arend behavior, not a bug.

```arend
\data D (n k : Nat) : \Set \elim k   -- two indices, only k matched
  | 1 => base
  | suc k' => stepD Nat (D n k')

\func f {n k : Nat} (x : D n k) : Nat \elim k, x
  | 1, base => n
  | suc k, stepD r rest => r Nat.+ f rest

-- WORKS: \elim k, rest exposes the function's clause.
\lemma f-step {n k : Nat} (r : Nat) (rest : D n k)
  : f (stepD r rest) = r Nat.+ f rest \elim k, rest
  | 1, base => idp
  | suc k, stepD r' rest => idp
```

**Subtler case — proving a property by induction on the parameter itself.** When the lemma is `f lambda = something` and you `\elim k, lambda`, the destructured `lambda` becomes `stepD r rest`, but **`f (stepD r rest)` still does NOT reduce inside this clause's body** — even though it looks like it should. The reducer needs `rest` to be the eliminated variable, not a sub-component of an eliminated variable.

```arend
-- FAILS: \elim k, lambda destructures lambda, but rest (sub-component) is
-- still a bound variable to Arend's reducer, not an eliminated one.
\lemma g=n-bad {n k : Nat} (lambda : D n k) : f lambda = n
  \elim k, lambda
  | 1, base => idp
  | suc k, stepD r rest =>
    pmap (r Nat.+) (g=n-bad rest) *> ...  -- Type mismatch: f (stepD ...) unreduced
```

The fix is to **factor the reduction step into a separate lemma** that `\elim`s the recursive arg, then chain through it:

```arend
\lemma g=n-good {n k : Nat} (lambda : D n k) : f lambda = n
  \elim k, lambda
  | 1, base => idp
  | suc k, stepD r rest =>
    f-step r rest                   -- f (stepD r rest) = r + f rest  (computation rule)
    *> pmap (r Nat.+) (g=n-good rest)  -- r + f rest = r + something
    *> ...                              -- chain to n
```

The pattern: **the reduction equation lives in a helper lemma whose proof eliminates the recursive arg as a parameter**. The induction proof then composes through this helper.

For data types whose constructors carry path-equality arguments (e.g. `compPart … (n = r1 + r2) …`), the helper's `\elim` may need to additionally destructure the path via `\case \elim path \with { idp => … }` to make the dependent indices rigid.

The same applies to `\sfunc` + `\peval`: `\peval f (C …)` fails with "Expression does not evaluate" if the call's recursive arg isn't an eliminated variable.

## 4. Records, classes, and fields

What I'd expect: records are dumb tuples, classes/typeclasses are a separate mechanism with resolution.

What Arend does (`tutorial/PartI/records.md`, `language-reference/definitions/records.md`, `classes.md`):

- `\record` and `\class` are the **same construct**; `\class` additionally enables instance inference for implicit arguments of that type.
- **Parameters are fields.** `\record Pair (A B : \Type) | fst : A | snd : B` — `A`, `B` show up as fields just like `fst`/`snd`. Anything in the parameter list is a field. You can `\extends` and implement them as you would any field.
- **Definitional eta** for records/classes: `\new R { | f => r.f | g => r.g }` is definitionally equal to `r`. Data types don't get this unless you opt in.
- **Classifying field**: the first explicit parameter of a `\class` is the classifying field (can mark others with `\classifying`, or disable with `\noclassifying`). An instance coerces automatically to this field's value. So `s : Semigroup` used where a `\Set` is expected becomes `s.E`; `m : Monoid Nat` used where a `Nat` expression is expected applies `m`'s operation only if the carrier field coerces to a function — more commonly this is what lets `Monoid Nat` match instance searches for `Nat`-carried structures.
- **Instance search**: implicit parameters typed by a class get filled by searching declared `\instance`s whose classifying field matches. No goal-directed unification cascade like Coq canonical structures — it's first-match-on-classifier.
- `\this` is an implicit parameter of type `R` available inside every field/function of record `R`. Usable only in argument positions.
- `\cowith` gives copattern-style definition: `\func zeros : Pair Nat Nat \cowith | fst => 0 | snd => 0`. Equivalent to `\new`, but can be the body of a `\func`.
- **Properties**: `\property p : P` requires `P` to be a proposition. Fields whose type is provably in `\Prop` are auto-promoted to properties unless marked `\field`. Properties don't compute — important for proof UX and performance.
- **Diamond handling**: `\extends A, B` where both extend `C` deduplicates `C`'s fields. But if `A` and `B` each independently define a field with the same name, you get two distinct fields and need qualified access `A.f {r}` vs `B.f {r}`. The algebraic-hierarchy advice is: avoid `\extends AbGroup, Monoid` for rings — use `\extends AbGroup { | mulMonoid : Monoid A }` to keep `*` and `+` in separate fields.
- **Arithmetic on a *concrete* record instance computes, and that breaks term-level tooling.** `ComplexField.*` is implemented by a formula on `re`/`im`, so `x * y` with either operand concrete normalizes to `\new Complex (x.re * y.re - x.im * y.im) (…)`. Consequences (all verified 2026-07-29, on what is now `Arith/Complex/FTA/Kneser.ard`): `equation.cRing` cannot prove even `x * y = y * x` at `Complex`; `rewrite` cannot match a pattern containing a meta under a `*`/`+`; and unification cannot infer an implicit that occurs only under one. **Keep such algebra symbolic: state the step over an abstract `{C : CRing}` (or make the operands explicit parameters of a helper) and instantiate.** Abstract class parameters are what keep a term neutral — this is a general reason a helper lemma over `{M : AbMonoid}` succeeds where the same proof inlined at the concrete instance fails. See **arend-prove** failure modes 15–17 for the diagnostics.

## 5. Universes, levels, and `\Prop`

What I'd expect: `Type i` with `i : Level`, cumulative or not depending on system.

What Arend has (`language-reference/expressions/universes.md`, `language-reference/definitions/level.md`, `tutorial/PartII/hom-levels.md`):

- **`\Type` has a single (predicative) level.** `\Type n` for a natural `n`; bare `\Type` is the *infinite* predicative level. Cumulative: `A : \Type n` ⟹ `A : \Type (suc n)`. There is **no second (homotopy) level argument** — `\Type \lp \lh` / `\Type p h` no longer exist (and `\lp`, `\lh`, `\oo`, `\oo-Type` are gone).
- **Homotopy levels are a separate family of universe keywords**, each carrying its own predicative level: `\Prop` (h-level -1, **impredicative** — no predicative level), `\Set n` = `\0-Type n` (h-level 0), `\n-Type m` (h-level n), and `\Type m` (untruncated). Homotopy cumulativity: `A : \Prop` ⟹ `A : \Set n` ⟹ … ⟹ `A : \Type n`. Placement: `\Prop : \Set 0`; `\k-Type p : \(k+1)-Type (p+1)`; `\Type p : \Type (p+1)`.
- **A truncated universe needs a *finite* predicative level.** Bare `\Set` / `\n-Type` errors with "Infinite level is not allowed here"; write `\Set 0` or use a level parameter. Only `\Type` may be bare. Consequently `\func x : \Type => \Type` is rejected (the infinite `\Type` isn't a member of itself — this is what blocks Girard-style paradoxes).
- **Level polymorphism is explicit and predicative-only.** No implicit `\lp`/`\lh`: a definition is polymorphic only if it declares named level parameters *after its name*: `\func id.{l} (A : \Type l) (a : A) => a`. Several params are comma-separated and independent: `\func f.{l1, l2} …`. A definition mentioning only bare `\Type` is **not** polymorphic (fixed at the infinite level). Homotopy level is not an abstractable axis.
- **Level arguments are passed with the same `.{…}` syntax**: `id.{2}`, `f.{\suc l}`, `f.{l1, l2}`. Usually inferred, so rarely written. The old `\levels p h` and positional-level forms are gone.
- **Level expressions**: atoms are non-negative numerals and named level parameters; operations `\suc l` and `\max l1 l2`.
- **Level of a data type** is the max over its **constructor-argument** type levels (a parameter not used by any constructor does not contribute). **Level of a record/class** is over its **unimplemented** fields. A definition that declares a level parameter `l` keeps that level: `\class C.{l} (A : \Type l) | …` gives `C.{l} X : \Type l`, even for a low-level `X` (e.g. `C.{5} Nat : \Type5`, not `\Type0`).
- Pi's homotopy level is the codomain's, not max; its predicative level is the max of domain and codomain.

## 6. `\use \level` — retroactively lowering the universe

This one has no Coq/Agda analogue I know of (`language-reference/definitions/level.md`, `data.md`, `functions.md`):

- If you prove a type has a lower homotopy level than was inferred, write a `\use \level <proof>` in its `\where` block. The type then *typechecks* as living in that lower universe (e.g., `Dec P : \Prop` even though it's declared with `| yes P | no (P -> Empty)` which is `\Set0` by default).
- The proof must target `ofHLevel (D p_1 ... p_k) n` for a constant `n`.
- For `\data`, this marks it **squashed**: pattern matching is only allowed when the result type lives in a universe ≤ the data's (or inside `\sfunc`, `\lemma`, `\use \level`, `\scase`).
- `\truncated \data ... : \Prop | ...` is the syntactic sugar for squashing into `\Prop`.
- Also: `\level T p` as a result-type annotation (`\lemma lem : \level A p => {?}`) convinces the checker `T` has the required level for the lemma/property/clause-omission.

## 7. `\func` / `\sfunc` / `\lemma` — evaluation semantics, not just typing

(`language-reference/definitions/functions.md`, `level.md`)

- `\func` — normal, reduces.
- `\sfunc` — never reduces; to force, use `\eval` at the call site. For performance on huge proofs whose computational content you don't want the typechecker to unfold.
- `\lemma` — result must be (provably) in `\Prop`; doesn't reduce; semantically a proof. Use for anything proof-ish where you don't want unfolding to bloat types.
- `\type` — a type alias that doesn't unfold (record-like transparency control).

Default to `\func`; switch to `\lemma` for proofs you'll cite many times and `\sfunc` when you hit typechecking slowness.

**`\have | x : T => v` can make `x` opaque to unification.** Inside a proof body, `\have` (or `\let`) with an explicit type annotation introduces `x` with type `T` whose value is `v`, but later goals that depend on `x` may *not* unfold to `v` — Arend can give errors like `Cannot solve equation: 1st expression: v, 2nd expression: x`. Two workarounds:

- Drop the type annotation: `\have | x => v`.
- Inline `v` directly at every use site (verbose but reliable).

The behavior is most painful when `v` is a long expression (e.g. `eps * RatField.finv (Ba + Bb + 1)`) that you want to abbreviate — but the abbreviation costs unification.

**It also applies to *unannotated* `\let`, and then it blocks implicit-argument inference.** A bound value is not unfolded when a lemma's implicit has to be read off it:

```arend
\let | w => ComplexField.negative (Complex.fromReal t * (b0 * conj bk))
     | cabs-w : cabs w = 1 => cabs_negative *> cabs_* *> …
```

fails with `Cannot infer parameter 'z' of definition 'cabs_negative'` — `cabs_negative {z} : cabs (negative z) = cabs z` cannot match `cabs w`. With a bound `pinv` the same opacity surfaces as a *type mismatch* between `zro < iS` and `zro < inv {pos#0 …}`, which reads like an instance-resolution problem instead.

The rule that follows: **never bind a value that a downstream lemma has to pattern-match against.** Three ways out, best first:

1. **Introduce it through an existential.** Have a helper lemma return `∃ (u : C) (P u) (Q u)` and destructure with `\case … \with { | inP (u, pu, qu) => … }`. `u` is then a genuine variable and every implicit downstream of it resolves.
2. **Make it an explicit parameter of a helper lemma** — `dir-cabs (b0 bk : Complex) (t : Real) (t>=0 : 0 <= t) : …`. Inside the helper everything is a variable; at the call site you pass the expression once.
3. **Write the expression out at each use site.** Always works, reads badly.

The same fix applies to `\Sigma`-returning helpers: building a tuple whose components each need an implicit inferred from a `cabs`/`pow` application tends to fail where two separate lemmas with explicit implicits succeed.

## 8. Pattern-matching data types with varying constructors

Not something Coq can express directly (`tutorial/PartI/datanproofs.md`, `language-reference/definitions/data.md`):

```arend
\data Vec (A : \Type) (n : Nat) \elim n
  | 0 => nil
  | suc n => cons A (Vec A n)
```

Constructor lists can **depend on how parameters pattern-match**. One clause can introduce multiple constructors. This also drives HITs: `| path-cons I-var => boundary-expr` clauses impose definitional equalities on a data type.

Strict positivity is enforced: the type cannot occur in a function-domain position nor inside `Path` endpoints.

`\cons f => body` defines a pattern-synonym-ish alias.

## 9. `Nat` / `Fin` / `Array` / `I` are subtly primitive

(`language-reference/prelude.md`)

- `Nat.+` and `Nat.-` reduce on **either side** (`n + 0 => n`, `0 + m => m`, `suc n + m => suc (n + m)`, `n + suc m => suc (n + m)`). You can't write these clauses in user code — pattern matching picks one direction. Prelude cheats.
- `Fin n` is a **subtype** of `Nat` (and of `Fin (suc n)`). `zero : Fin (suc n)`, `suc x : Fin (suc n)` when `x : Fin n`. No separate constructor names.
- `Array A` is a **record** with `len : Nat`, `A : \Type`, `at : Fin len -> A`; it also has `nil` and `::` constructors. You can pattern-match like a list *and* index like a function, simultaneously. `Array A n` is equivalent to a length-`n` vector. `DArray` generalizes to dependent fibres.
- `I` has constructors `left`, `right` and a hidden "connecting" constructor. **Pattern-matching on `I` is forbidden**. Use `coe`, `coe2`, `squeeze`, `squeezeR` instead. `coe (\lam x => A) a i => a` when `x` doesn't occur free in `A` (key reduction rule).
- `iso` implies univalence and is not definable in user syntax.
- `Path.inProp` postulates proof irrelevance and has no body.

## 10. Coercions

(`language-reference/definitions/coercion.md`, `records.md`)

- `\use \coerce toT (x : S) : T | ...` in a `\where` defines a coercion `S → T` (last param is source). For target-direction, the result type defines the target.
- `\coerce` annotation on a record field or data constructor promotes it to a coercion.
- A class's classifying field gives an automatic coercion to that field's type.
- Coercion into a function type partially applies.
- Because coercions fire silently, if typechecking reports a surprising type, check for an inserted coercion before blaming unification.
- **Hoist explicit embeddings to `\use \coerce` when you'd otherwise repeat them.** If a `.ard` file inserts `fromReal x` / `wrap x` / `inj x` at every call site, define `\use \coerce fromReal (x : Real) : Complex => …` once inside the target record's `\where`, and the elaborator silently inserts it everywhere. The 2026-05-15 refactor moved the `Real → Complex` embedding to a `\use \coerce` inside `Complex \where`; downstream `I * fromReal x` instances throughout `Arith/Trig/Real.ard`, `Arith/Trig/Complex.ard`, `Arith/Complex/Euler.ard`, and `Arith/Complex/Norm.ard` collapsed to `i * x`. The coercion is at `Arith/Complex.ard:23`; the accompanying `\open Complex \using (iunit \as i)` appears at the top of each of those files.

## 11. Implicit arguments, `\meta`, and the tactic story

(`language-reference/definitions/metas.md`, `functions.md`, `standard-tactics/*`)

- Implicits: `{A : \Type}`. In call sites, curly-brace application; in patterns, either give `| {A}, x => ...` or omit implicit patterns.
- `_` in an argument = infer; `_` as a parameter name = "don't care" (repeatable).
- Inference *across constructor-injectivity* works: `{n m : Nat} (p : suc n = m)` → both can be inferred. `{n : Nat} (p : n + n = 3)` → cannot, since `+` is not a constructor.
- Goals: `{?}` is a hole.
- **No tactic language in the core.** "Tactics" are `\meta`s — macros that expand into Arend expressions, type-checked post-expansion. Defined either in Arend (`\meta f x => e`, pure substitution) or in Java (the interesting ones, shipping as the standard library's `Meta` modules).

For the catalog of metas (`rewrite`, `ext`, `simplify`, `equation`, `cong`, `linarith`, `cases`, `mcases`, `unfold`, `assumption`, `in`, `at`, `run`, `$`, `#`, `repeat`, `using`, `defaultImpl`, …), their semantics, common failure modes, and a decision table, see the **arend-prove** skill.

## 12. Syntax quirks I keep forgetting

- All keywords start with `\`: `\func`, `\data`, `\class`, `\record`, `\let`, `\with`, `\elim`, `\case`, `\lam`, `\Pi`, `\Sigma`, `\new`, `\this`, `\where`, `\open`, `\import`, `\hiding`, `\using`, `\instance`, `\cowith`, `\scase`, `\sfunc`, `\lemma`, `\property`, `\field`, `\truncated`, `\coerce`, `\level`, `\eval`, `\extends`.
- Infix: `\infix`, `\infixl`, `\infixr` + priority 1–9. Prefix application of an infix with backticks: ``x `op` y``.
- Comments: `-- line`, `{- nestable block -}`.
- `()` pattern — absurd case, RHS omitted. Often nested: `| fsuc (fsuc (fsuc ()))`.
- `\as x` binds an as-pattern; in `\case`, `\as x \return T[x]` is the dependent idiom.
- Dot notation: `x.f` works **only** if `x` is a variable whose type is a literal record/class name. For general expressions use `Record.f {x}`.
- `__` (double underscore) — implicit lambda slot, enables sections like `__ + 1`.
- `\open M`, `\open M (f, g)`, `\open M \hiding (f)`, `\open M (f \as f')`, `\open M \using (f \as f')`. `\import` bundles visibility. **`\open M \using (Long \as short)` is the idiom for aliasing a hot identifier file-locally** — e.g. `\open Complex \using (iunit \as i)` at the top of `Arith/Trig/Real.ard:34` (and `Arith/Complex.ard:19`, `Arith/Trig/ArcTan.ard:49`, `Arith/Trig/Complex.ard:19`, `Arith/Complex/Euler.ard:14`, `Arith/Complex/Norm.ard:38`) lets the body write `i * x` everywhere instead of `Complex.iunit * fromReal x`. Reach for it whenever a long-named constant is used more than ~3 times.
- `\let` bindings are non-recursive and sequential (each sees the previous).
- `\where` is a module; `f.g` accesses it from outside. **But `\private \lemma helper` inside a `\where` block makes `helper` invisible outside**, and because Arend's name resolution then falls back to treating `f.helper` as field access on `f`-the-value, the error is misleading: `Type mismatch: Expected type: a class. Actual type: <f's body type>`. If you want sibling lemmas to share helpers, either drop `\private` (so the helper is reachable as `parent.helper`) or hoist to file-level `\private \lemma` at module scope — arend-lib usually does the latter.

## 13. Termination checker surprises

(`language-reference/term-checker.md`)

- Lex order on declared parameters; size-change for mutual recursion.
- **Only top-level `\elim` / `\with` eliminations count.** A decreasing `\case` in the body is invisible to the checker.
- If the checker rejects a legitimately decreasing call, factor the recursion out to a helper with `\elim`.

## 14. The "reasoning-about-types" payoff

Things that are axioms or advanced features in other systems but fall out naturally here:

- **Function extensionality** is provable (use paths through the interval).
- **Propositional univalence** (`A ↔ B ⟹ A = B` for `A B : \Prop`) is provable; **full univalence** comes from `iso`.
- **`\new R {...} ≡ r`** eta gives "record refactoring" for free definitionally.
- `path (\lam i => p @ i) ≡ p` eta for `@` is definitional.

For deciding which meta closes an equality goal, see the **arend-prove** skill.

## 15. ExUpperReal idiosyncrasies (and why generic Monoid/Semiring lemmas don't always apply)

`ExUpperReal` represents "real numbers possibly equal to +∞". Its arithmetic is fragile in three ways that catch out generic algebraic reasoning:

- **`ExUpperReal` is NOT a `PosetSemiring`**, despite having `+`, `*`, and `<=`. arend-lib instantiates `ExUpperRealAbMonoid : BiorderedLatticeAbMonoid` (for `+` with order) and `ExUpperRealSemigroup : CSemigroup` (for `*`) as *separate* class instances. There's no unified `PosetSemiring ExUpperReal`. **Consequence**: `PosetSemiring.pow_<=-degree`, `PosetSemiring.pow<=1`, and similar lemmas that need both `+` and `*` aligned don't apply. (Those live on `\class PosetSemiring` — `Algebra/Ordered.ard:236`, with `pow_<=-degree` at line 285 — and `ExUpperReal` is not an instance of it.) If you need such a lemma over ExUpperReal, prove the analog manually; as of 2026-07-28 the library has no ExUpperReal-specific `pow_<=-degree`, so there's nothing to copy.

- **`ExUpperRealSemigroup.ide-left`/`ide-right` are CONDITIONAL on `x >= 0`**. Unlike `Monoid.ide-right : x * ide = x` (unconditional), the ExUpperReal versions carry a non-negativity hypothesis:

  ```arend
  -- Arith/Real/UpperReal.ard:336, :341
  \protected \lemma ide-left  {x : ExUpperReal} (x>=0 : 0 <= x) : fromRat 1 * x = x
  \protected \lemma ide-right {x : ExUpperReal} (x>=0 : 0 <= x) : x * fromRat 1 = x
  ```

  Always supply the proof when calling them. `ExUpperRealSemigroup.pow_>=0` (`Arith/Real/UpperReal.ard:518`) is the usual source — see `Arith/Log.ard:77` and `Arith/Trig/ArcTan.ard:96` for the idiom `ExUpperRealSemigroup.ide-left ExUpperRealSemigroup.pow_>=0`.

- **`ExUpperRealSemigroup.pow` is a custom `\protected \func`, NOT `Monoid.pow`** (`Arith/Real/UpperReal.ard:506`). They're different functions of the same shape but with different definitional behaviour. So `Monoid.pow_+`, `Monoid.pow_ide`, `pow_*-comm`, etc. don't apply to `ExUpperRealSemigroup.pow`. The custom pow's definition `pow x (suc n) = pow x n * x` does reduce, so basic induction works fine. It ships with `rat-pow`, `pow_<=`, and `pow_>=0` (`Arith/Real/UpperReal.ard:510–518`); anything beyond those (`pow_+`, `pow_ide`) you prove yourself.

## 16. `RealNormed.norm` returns ExUpperReal but is defined as `abs : Real → Real`

`Topology/NormedAbGroup/Real.ard` defines `RealNormedAbGroup.norm := abs`, where `abs : Real → Real` is implicitly coerced (somehow — there's no explicit `Real → ExUpperReal` `\use \coerce`; the class field's type `ExUpperReal` is reached by a `join (Real.fromRat r) (negative (Real.fromRat r))` form). The practical consequence: when proving `RealValuedRing.norm (Real.fromRat r) <= ExUpperReal.fromRat c`, the goal unfolds to `join (fromRat r) (negative (fromRat r)) <= ExUpperReal.fromRat c` — comparing a Real value against an ExUpperReal value.

**The bridge** to use:
1. `LatticeAbGroup.abs-ofPos : 0 <= x -> abs x = x` — collapse the `join`/`negative` form to `x` when `x >= 0`.
2. `RealAbGroup.abs-rat : abs (Real.fromRat r) = Real.fromRat (RatField.abs r)` — push abs into Rat.
3. `Real.<=-upper : x <= y (in Real) <-> x <= y (in ExUpperReal)` — bridge the two `<=`.
4. `rat_real_<= : a <= b (in Rat) <-> Real.fromRat a <= Real.fromRat b (in Real)` — bridge to Rat.

(Locations: `abs-ofPos` on `LatticeAbGroup`, `abs-rat` at `Arith/Real.ard:302`, `Real.<=-upper` and `rat_real_<=` in `Arith/Real.ard`.)

Composing these turns any `RealValuedRing.norm (fromRat r) <= ExUpperReal.fromRat c` goal into a pure Rat inequality `RatField.abs r <= c`.

Worked instances in the current tree: `Arith/Log/Properties.ard:91` (`A.norm_fromRat <=∘ =_<= (pmap ExUpperReal.fromRat (RatField.abs-ofPos …))`) and `Topology/StoneCStarAlgebra.ard:251`. When the target is already an `ExUpperReal` bound, the shorter `norm_fromRat <=∘ ExUpperReal.<=-rat.1 …` form often replaces the full four-step chain — see `Arith/Trig/ArcTan.ard:95`.

## 17. `makeRat` doesn't reduce on `suc <var>` — case-split to expose

`ratio nom (suc m)` calls `makeRat nom (suc m) ...`, which eliminates its `denom` into two clauses (`Arith/Rat.ard:64`):

```arend
\func makeRat (nom : Int) (denom : Nat) (denom/=0 : denom /= 0) : Rat \elim denom
  | 1 => nom                                  -- literal return, no reduction
  | denom => makeRat' nom denom denom/=0      -- catch-all; computes via `reduce`
```

If `m` is a variable, `suc m` matches neither clause definitionally — the elaborator can't tell whether `suc m = 1`, so it can't commit to clause 1 or fall through to the catch-all. As a result, `ratNom (ratio 1 (suc m))` does NOT reduce to `pos 1`, even though it equals `pos 1` propositionally. `simplify` and `linarith` can't see through.

**Fixes, cheapest first:**

1. **`makeRat.simp`** (`Arith/Rat.ard:77`) — `makeRat nom denom denom/=0 = makeRat' nom denom denom/=0`, already proved for every `denom`. Rewriting with it discharges the split for you.
2. **`mcases {makeRat}`** — mirrors `makeRat`'s own clause structure; see `Arith/Rat.ard:99` (`signum_nom`) for a live use.
3. **`\elim m` / `\case \elim m`** — the manual version. In each branch `makeRat` reduces (clause 1 in the `m = 0` case, the catch-all otherwise), `ratNom`/`ratDenom` become concrete, and the residual goal is usually linarith-soluble.

## 18. `\private` inside `\where` hides the def from sibling lemmas — and the error message is misleading

`\where` IS a module, and `parent.helper` IS valid syntax for accessing nested definitions from outside the parent. What goes wrong is the combination `\private \lemma helper` *inside* a `\where`: `helper` becomes invisible outside the parent, AND because name resolution falls back to treating `parent.helper` as field access on `parent`-the-value, the error has nothing to do with privacy:

```
[ERROR] Type mismatch:
  Expected type: a class
  Actual type: <parent's equation type>
  In: parent
```

If two sibling lemmas need to share helpers nested in a `\where`, either:

- **(a) Drop `\private`** so the helpers are publicly accessible as `parent.helper`. Fine when the helpers genuinely belong to the parent's namespace.
- **(b) Hoist to file-level `\private \lemma`** at module scope. This is the more common arend-lib idiom.

(Cross-references from a sibling inside the same `\where` to another sibling at the *same* depth work fine without qualification — it's only the *outside-the-where* access that needs the public visibility or the hoist.)

## 19. Block comments `{- ... -}` nest — LaTeX in doc comments can swallow the whole file

Arend's block comment `{- ... -}` is **nestable**, so the parser counts opening and closing braces. Any `{-` inside a block comment opens a *new* nested comment that needs its own `-}`. In LaTeX-heavy doc comments this fires accidentally:

```arend
{- | Cauchy–Schwarz: $(\Re x \cdot \Re y)^2 \leq |x|^2 \cdot |y|^2$, where $x^{-1}$ denotes the inverse. -}
\lemma CS_<= ...
```

The `{-1}` inside `x^{-1}` opens a nested comment. The closing `-}` of the doc block then matches the inner nesting, leaving the *outer* block unclosed — so the entire rest of the file becomes one giant comment. Symptoms:

- Definitions below the comment "disappear" (typechecker reports them as unresolved from elsewhere in the project, or simply not present).
- The file "passes" with zero diagnostics on the definitions in it, because none of them were parsed.
- Wildly distant errors like "no such module" or "definition X not found" appear in *other* files that depended on the swallowed ones.
- **Within one file, the report is `Cannot resolve reference 'X'` at the *consumer*** — a line 20 lines below a definition of `X` that is plainly visible in the source. Nothing points at the comment.
- The swallowed range ends at the *next* `-}` in the file, not at EOF, so which definitions vanish looks arbitrary: three consecutive ones can disappear while a fourth further down survives.
- `-r --serialize` changes nothing (it is not a cache problem) — don't spend a rebuild on that guess.
- **Fastest confirmation: `arend -ss <name>` returns `No matches` for a definition that is on screen.** If the symbol index cannot see it, the parser never saw it either. (Contrast with the genuine index lag for brand-new *modules*, noted in **arend-prove**.)

A real instance: `{- | The Kneser factor $q = 1 - 3^{-2n^2 - n}$ … -}` — the `{-` in `^{-2` swallowed `kneser-q`, its two bound lemmas, and two more helpers.

**Remedy:** after editing any doc comment, run `grep -rn` for the three forms and inspect every hit inside a doc comment:

```bash
grep -rn '{-' arend-lib/src/
grep -rn -- '-}' arend-lib/src/
grep -rn '{-}' arend-lib/src/    # rarer: empty nested comment from `{-}` in math
```

Any occurrence of `{-`, `-}`, or `{-}` *inside* a doc comment (i.e. not the comment's own opening/closing delimiters) needs a space inserted to break the token: `{ -`, `- }`, `{ - }`. So `x^{-1}` becomes `x^{ -1}`, and `\{-1, 0, 1\}` becomes `\{ -1, 0, 1 \}`. Do this defensively after every doc-comment edit — the failure mode is silent and the diagnostic surfaces far from the cause.

(Line comments `-- | ...` do not nest and are immune. Prefer `-- |` for one-liners that don't contain LaTeX braces; reserve `{- | ... -}` for multi-line docs and accept the audit cost.)

## 20. Extending records and closing inherited fields (`\extends` / `| F => v`)

> **Provenance.** The running example below (`RatAnalytic`, `BoundedDiskRatAnalytic`, `disk-open-aux`, `eval-IsSeriesSum`, `disk-incl`, `radius-loc`) came from `Analysis/RatAnalytic.ard`, which commit `8363cdac2` "Refactor RatAnalytic" **deleted** — the development was dissolved into plain functions in `Analysis/PowerSeries.ard` and `Analysis/PowerSeries/Derivative.ard`, with no `\record` surviving. Don't go looking for those names; they aren't in the tree. The *library types* the section cites are all still live: `CoverMap`, `SetHom` (with `\coerce func`, `Set/SetHom.ard:4`), `StronglyCauchyMap`, `PrecoverSpace.open-char`, `OpenCompleteCoverSpace`, `RegularPreuniformSpace`, `func-cover` / `func-weak-cauchy` / `func-cauchy`. The lessons are general `\extends` behaviour and still apply.

When making your record a descendant of another (e.g. `\record RatAnalytic ... \extends CoverMap`) and supplying defaults for the parent's fields, several non-obvious things hold:

- **Field order is strict.** All `|`-fields (including assignments like `| InheritedField => value`) must come *before* any `\func` / `\lemma` methods in the record body. Mixing them yields confusing "field implementation cycle" or "cannot resolve reference" errors that never mention ordering. If you see `Cannot resolve reference 'foo'` for a method `foo` defined *below* in the same record, ordering is the first thing to check.

- **`| InheritedField => value` closes the parent's field.** This is the canonical pattern when extending: the child supplies a default value for what the parent left abstract.

  - `\override Field : NewType` is distinct — it changes the field's *type* (must be a subtype), not its value. Use it (e.g. `\override Dom : RegularPreuniformSpace`) when narrowing the inherited type; use `| Field => v` to supply a default.

- **Field-default expressions can't forward-reference methods of the same record.** A `| Dom => OpenCompleteCoverSpace disk-open` where `disk-open` is a `\lemma` defined later in the body fails with `Cannot resolve reference 'disk-open'`. Even reordering the lemma above the default may not help — the elaborator processes field defaults before method bodies are typechecked.

  **Workaround**: extract the helper to a top-level `\func` parameterized by what would have been the implicit `\this`-fields:
  ```arend
  \lemma disk-open-aux {A : RealBanachAlgebra} {r : ExUpperReal} : A.isOpen ... => ...

  \record RatAnalytic (A : RealBanachAlgebra) \extends CoverMap {
    | radius : ExUpperReal
    | Dom => OpenCompleteCoverSpace (disk-open-aux {A} {radius})
    ...
    \lemma disk-open : A.isOpen ... => disk-open-aux {A} {radius}   -- thin wrapper
  }
  ```

- **Implicit arguments are *not* inferred from field-default context.** Even though `A` and `radius` are record fields in scope at the default site, writing `| Dom => OpenCompleteCoverSpace disk-open-aux` fails with `Cannot infer parameter 'A'`. Must be explicit: `disk-open-aux {A} {radius}`. The default-site is not an ordinary call-site for purposes of metavariable resolution.

- **Don't redeclare the parent's class parameters in a descendant.** `\record Child (A : T) \extends Parent` where `Parent` already has `(A : T)` produces `Field 'A' is already defined in super class at Parent.A`. Just `\record Child \extends Parent { ... }` — `A` is inherited.

- **`\coerce` on a field propagates through `\extends`.** `\record SetHom (Dom Cod : BaseSet) | \coerce func : Dom -> Cod` makes any descendant of SetHom (transitively: ContMap, PrecoverMap, CoverMap) implicitly callable as a function. So `\record RatAnalytic \extends CoverMap` lets `(myAnalytic : RatAnalytic) s` desugar to `myAnalytic.func s` for free. Useful for "this record *is* a function with extra structure" — but it also means once you `\extends`, any field of the inherited `func` name is the coercion target. Existing `\func func` methods in the same record clash; rename them (e.g. `eval`).

- **"Field implementation cycle" can be a false positive from inherited default chains.** CoverMap has `func-weak-cauchy F => ...func-cover F...`, StronglyCauchyMap has `func-cauchy F => ...func-weak-cauchy F...`. Setting `func-cover` in a descendant from an externally-constructed CoverMap triggers `Field implementation cycle: func-cover - func-cauchy - func-weak-cauchy` even though the dependency is acyclic — the chain is `func-cauchy ← func-weak-cauchy ← func-cover`, all forward.

  **Workaround**: either extract the construction to top-level so it doesn't go through the descendant's `\this`, or close *all* the related defaults at once (`| func-cover => ...`, `| func-weak-cauchy => ...`, `| func-cauchy => ...`). The error message doesn't say which approach is needed; pick the smaller diff.

- **Vacuous-hypothesis discharges via empty-pattern `\case`.** For an inherited field whose hypothesis is provably empty in a specific instance (e.g. `radius.IsBounded` when `radius = top` — `top.U q` is `∃ over Fin 0`, empty), supply the field body with a pattern-match that ends in an empty case:
  ```arend
  | radius-loc => \lam rb _ _ => \case rb \with {
      | inP (B, top-U-B) => \case top-U-B \with { | inP (j, _) => absurd j }
    }
  ```
  The outer `inP (B, top-U-B)` destructures the truncation; the inner `\case top-U-B \with { | inP (j, _) => absurd j }` destructures the empty `∃` and absurd-eliminates `j : Fin 0`.

- **Total-Set domain replaces per-point-with-proof signatures.** When a record extends `CoverMap` with `Dom = Total S` for `S : Set X` a predicate, the inherited `func : Dom → Cod` is total on `Total S = \Sigma (x : X) (S x)`. Per-point access becomes `func (x, h)` (with the coercion: `myInstance (x, h)`). If you previously had a per-point `\func func {x} (h : S x) : Y`, rename it — both can't be `func`.

- **Methods invoked inside a class body need an explicit instance qualifier when the receiver is the same parent class.** Calling `PrecoverSpace.open-char.2` (a method of `PrecoverSpace`) inside a `BoundedDiskRatAnalytic` body whose `Dom` is a `PrecoverSpace` doesn't pick up `Dom` as the implicit receiver — instead the lambda parameter `s` in the callback ends up typed `E {?this}` where `?this` is unresolvable. The fix is to qualify with the actual instance value:
  ```arend
  -- doesn't work (lambda's s gets generic E {?this}):
  PrecoverSpace.open-char.2 \lam {s} Ufs => ... s.2 ...

  -- works (lambda's s now resolves to Total ...):
  (OpenCompleteCoverSpace (disk-open-aux {A} {radius})).open-char.2 \lam {s} Ufs => ... s.2 ...
  ```
  Same trick on the inner `open-char.1` if used inside the same proof. Without the qualifier, `s.1` / `s.2` projections fail with "Expected: sigma, Actual: E" — there's no hint that the fix is at the surrounding call.

- **`|`-style field defaults cannot reference class `\lemma` / `\func` methods, by any syntax.** Inside a `\record`'s `| Field => body`:
  - Bare names of class methods (`eval-IsSeriesSum`, `disk-incl`, …) **don't resolve** ("Cannot resolve reference"), even when those methods are defined earlier in the same record. By contrast, `\lemma` / `\func` bodies *can* reference other class methods by bare name.
  - `\this.method` and `ClassName.method {\this}` **resolve** but both trigger an immediate `Dependency cycle` through the class itself — arend's dependency tracker treats them as the field default depending on the class via `\this`.

  The workaround is to **inline the method body** in the field default. The only references that work cleanly from `|`-defaults are:
  - bare field names (`coef-Rat`, `ps-conv`, …) — fields are visible by bare name from defaults.
  - top-level helpers (e.g. `disk-incl-aux {A} {radius}` instead of `\this.disk-incl`).

  Concretely, the func-cont proof in `BoundedDiskRatAnalytic` needed:
  ```arend
  -- doesn't work — bare resolution fails for `eval-IsSeriesSum`:
  ... A.limit-unique limit-isLimit (eval-IsSeriesSum s.2) ...

  -- doesn't work either — \this triggers cycle through the class:
  ... A.limit-unique limit-isLimit (\this.eval-IsSeriesSum s.2) ...

  -- works — inline eval-IsSeriesSum's body, using only fields and externals:
  ... A.limit-unique limit-isLimit (seriesConv-sum (absConv-isConv (...ps-conv norm>=0 s.2...))) ...
  ```
  Same goes for `disk-incl` (replace with the top-level `disk-incl-aux {A} {radius}`).

These pitfalls compound: a single naïve `| InheritedField => method-defined-later` line can trigger any of forward-reference-failure, implicit-arg-failure, *and* implementation-cycle errors, none of which name the underlying ordering/scoping issue. When debugging a record refactor, audit field-default expressions for: (a) only-fields-and-top-level-functions referenced, (b) all implicit args spelled out, (c) `|`-block ends before the first `\func`/`\lemma`, (d) sibling-class method calls (like `open-char`) are qualified with the receiver instance.

## 21. Quick decision table when stuck

| Symptom | First thing to try |
|---|---|
| Goal has `f x` where `f` was defined with `\case` | Rewrite `f` to use `\elim` |
| Termination error on obvious recursion | Move `\case` up to `\elim` in the signature |
| "Cannot match on `idp`" | Ensure one side of the equality is a variable bound in the match context and doesn't appear in the other side |
| Implicit arg not inferred from `p : f n = m` | `f` is not a constructor — provide it explicitly or restructure |
| Proof unfolds into a mess | Switch `\func` → `\lemma` |
| Typechecker is slow on a big proof | Switch `\func` → `\sfunc`, use `\eval` at call sites only as needed |
| Need the connection between a case scrutinee and its matched value | `\case e \as x, idp : e = x \with { ... }` (or see arend-prove for `cases`/`mcases`) |
| Data type inferred in too-high universe | `\use \level` with a level-proof in `\where` |
| Need to squash into `\Prop` | `\truncated \data ... : \Prop` |
| Want definitional equality for a derived def | `\type` instead of `\func` (or leave it as `\func`, which reduces) |
| Equality / algebra / arithmetic goal | See the **arend-prove** skill |
