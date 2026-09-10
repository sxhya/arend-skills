---
name: arend-formalize
description: Procedural workflow for turning an informal mathematical statement into a typechecking Arend definition/lemma. Use when the user gives an informal claim and asks for a formalization, or when starting a new .ard file from a math text. Covers what to read first, which CLI tools to reach for, and the order of operations from "I have an idea" to "only GOAL errors remain".
user-invocable: true
---

# Arend formalization workflow

A staged process for going from an informal claim to a typechecking Arend statement. The three phases are interleaved in practice — but resist the temptation to start writing `.ard` before phase 1 is done.

For language-level surprises while writing the statement, see the **arend-quirks** skill. For closing equality/algebra goals once the statement compiles, see the **arend-prove** skill.

---

## Phase 0 — Start the daemon

> **`--serialize` exists and is opt-in.** Verified against `ConsoleMain.parseArgs` on the current tree: the flag is registered as
> *“after typechecking, persist typechecked modules as .arc binary caches; **without this flag, no .arc files are written**”*.
> There is **no `--no-serialize`** — passing it fails with `Unrecognized option: --no-serialize`.
> An earlier revision of this skill claimed the opposite (removed flag, serialization on by default, `--no-serialize` opt-out).
> That was wrong in both directions; every recipe below has been corrected. **Writing `.arc` requires `--serialize` on the call
> that does the typechecking** — and for a daemon-served call that means on the `-d` bootstrap (see below).

The edit loop is dominated by JVM startup and library load, not by typechecking your edit. The fix is the **daemon**: a long-lived JVM holding a warm `ArendServer`. Start it once per session:

```bash
cd /path/to/arend-lib && arend . -d --serialize
```

Bootstrap runs the ordinary typecheck pipeline once over the whole library — with exactly the flags `-d` forwarded
(`-s -e -m -c -r --serialize`) — and then idles. **Every subsequent `arend` call in that library auto-routes to it**
— no flag needed on the client side — so a scoped typecheck comes back in seconds instead of minutes.
`--daemon-refresh` re-submits that same frozen bootstrap argv against the warm context (`DaemonServer`'s `refresh`
op literally replays `ctx.bootstrapArgs`), which is how source edits get picked up.

| Command | Effect |
|---|---|
| `arend . -d` | start the daemon (needs a `LIBRARY` anchored to an `arend.yaml`, or `-s <dir>` for a synthetic library) |
| `arend . --daemon-ping` | `IDLE` or `BUSY`; `[ERROR] ping: no daemon running…` if there is none |
| `arend . --daemon-status` | dump state |
| `arend . --daemon-refresh` | re-run the `-ai` bootstrap on the warm context so source edits are picked up |
| `arend . --daemon-stop` | stop it |
| `arend … --no-daemon` | force in-process execution for this one call |

### Flags the daemon eats

This is the part that bites. On a daemon-served call these are **parsed but silently ignored**, locked to the daemon's bootstrap values:

```
-L   -s   -e   -m   -c   -r   --serialize
```

(That is `LockedFlags.NAMES` verbatim. `--slow-warn` is **not** a flag on the current tree either — an older
revision listed it here and it hard-errors.)

`--serialize` being locked is the second consequential entry: **a daemon started without `--serialize` will never write an
`.arc` file, no matter how many served calls pass the flag.** If you want warm caches, bootstrap with
`arend . -d --serialize` — `-d` forwards `-s -e -m -c -r --serialize` to the child JVM, and the child's copy is the one
that counts. `--daemon-status` prints the locked set actually in force, so it settles the question.

`-r` being in that list is the consequential one: **a plain `arend -r <Module>` against a live daemon does not recompile anything.** To actually force a from-source pass you must either add `--no-daemon`, or `--daemon-stop` first.

These are rejected outright inside a daemon-served command (not ignored — hard error): `-i`, `-d`, `--daemon-stop`, `--daemon-ping`, `--daemon-status`, `--daemon-refresh`.

### Binary caches (`.arc`) under the current CLI

| | Behaviour |
|---|---|
| **Loading** existing `.arc` | On by default. `-r` / `--recompile` turns it off for one run — *but see the daemon-eats-`-r` note above*. |
| **Writing** `.arc` after typecheck | **Off by default.** `--serialize` turns it on. There is no `--no-serialize`. On a daemon-served call the flag is locked to the `-d` bootstrap's value. |

**When the cache itself is the suspect.** After a class-hierarchy refactor, cached expressions referencing the old internal terms still deserialize cleanly and report errors that don't exist in the sources. Symptoms:

- Bogus type mismatches naming what looks like the same type twice (`Rat` vs `Arith.Rat.Rat`).
- Errors on definitions you never touched, which vanish under `-r`.
- A Java stack trace out of the (de)serializer rather than a typecheck diagnostic.

Recover with **`arend --no-daemon -r --serialize <Module>`**. All three parts matter:

- `--no-daemon` because a daemon-served `-r` is a no-op;
- `-r` because it skips *loading* the caches;
- `--serialize` because otherwise the clean build **writes nothing** and the poisoned `.arc` is still on disk for the next run.

The `-r --serialize` pairing is therefore *required*, not obsolete: `-r` reads past the bad cache, `--serialize` replaces it.
Drop `--serialize` and you get a correct one-off answer followed by the same bogus errors on the very next invocation.

### The daemon has its own failure mode: poisoning

A resolution error during a daemon run can leave the warm server in a state where importers deny a definition that is plainly visible in the sources. **In-daemon `-r` does not recover it** (it's ignored — see above); neither does `--daemon-refresh` reliably. The only reliable fix is:

```bash
arend . --daemon-stop && arend . -d
```

Related, and now fixed: the daemon used to report stored resolver errors only on the *first* run, so a warm re-run of a still-broken module could exit 0 with no diagnostics (arend-lang/Arend#138, fixed 2026-08-08). On a current build, warm-run exit codes are trustworthy again and errors print after the banner. If you are on an older build and a warm run looks suspiciously clean, re-check with `--no-daemon`.

### `-ai` — the agent-oriented pipeline

> **Not present on the current tree (verified 2026-09-10).** `-ai` is not a registered option in
> `ConsoleMain.parseArgs` on either `staging` or `cliDaemon-9`, no source file writes a `.sig/` mirror, and
> `TypecheckPipeline` contains no symbol-index refresh. Passing `-ai` hard-errors with `Unrecognized option`.
> The daemon's bootstrap is a plain typecheck of the library, **not** an `-ai` pipeline. Treat this section as a
> description of an unmerged/planned feature; do not build a workflow on it until `-ai` shows up in `--help`.

As designed, the daemon's bootstrap would *be* `-ai` and `--daemon-refresh` would re-run it; standalone `arend . -ai [MODULE[:DEF]]`. Four stages:

1. **Name resolution (suggest-only)** — for each unresolved short name, prints a `Candidates for 'X' at …` block with each candidate's `library::module:longName`, the qualified name to splice in, and the required import. It never rewrites your sources; you paste the pick yourself.
2. **Typecheck** of the in-scope modules.
3. **Signature mirror** — writes signature-only views of every typechecked module to `<library>/.sig/<module>.ard`, with bodies and field implementations replaced by `{?}`. Definitions with typecheck errors become `-- skipped: <name> (typecheck errors)`, so **`.sig/` only ever contains verified declarations** — that is what makes it safe to read as ground truth.
4. **Symbol index refresh** — rebuilds the on-disk index behind `-ss` / `-fu` / `-ch` / `-sc`.

Output is quiet by default (verbose narration goes to a per-invocation log); `--no-quiet` puts it back on stdout.

**Run every call from inside the library directory.** `cd arend-lib` first: the typecheck workflows and the retrieval flags (`-ss`, `-ps`, `-fu`, `-ch`, `-sc`) all expect the library in scope, and invoking them from the parent dir crashes with `NullPointerException: ctx.outputRouter is null` instead of a clean "no library in scope" error.

---

## Phase 1 — Before formalizing: orient yourself in arend-lib

Goal: figure out whether the prerequisites already exist, whether someone has already done this, and what names/classes to reuse. Cheap reads first, expensive reads last.

1. **Start with the retrieval flags, not the sources.** `-ss` / `-ps` / `-fu` / `-ch` / `-sc` query the loaded library without typechecking it, and they answer "does this exist / what shape is it / who uses it" far more cheaply than reading `.ard` files. Work out *which module* owns the concept from their qualified results, and only then open sources.

2. **When you do open a source file, read the signature, not the body.** Definitions in arend-lib are signature-first: the `\func` / `\lemma` / `\class` line plus its result type is what you need to decide whether something applies. Bodies are large, meta-heavy, and rarely relevant to an existence question — skip past them.

   **Exception — scout the idiom before extending an existing module.** Once you know *which* module you are adding to, read the bodies of the three or four nearest existing definitions and write down how they are *proved*, not what they are called. No name you can guess will tell you "this module reasons via `AbMonoid.FinSum` over `SigmaFin`/`PairsFinSet`, not via `BigSum` over `\new Array`" — and picking the wrong one costs an order of magnitude in proof length that no later cleanup recovers. This is the one place where reading bodies is the cheap move.

3. **For name-pattern lookup, use `arend -ss <pattern>` (symbol-search).** Fast, cached on-disk index. Becomes more useful as you internalize arend-lib's naming conventions (e.g., `+-comm`, `*-assoc`, `inv-r`, `iso->equiv`).

   **Default is case-insensitive substring on short names — *not* regex.** Reaching for `.*` or `\|` means you guessed wrong:
   - `-ss BigSum contains=<=` for "names containing both substrings" (AND).
   - `-ss A -ss B` for "names matching either pattern" (OR via multiple flags).
   - `-ss "glob:Big*<="` for genuine wildcards, `-ss "re:^abs.*"` for raw regex, `-ss "hb:isP"` for camel/dash-fuzzy, `-ss "glob:exact-name"` (no wildcard chars) for an exact match. **The `eq:` mode was removed** — using it now errors out with a fix-it pointing at `glob:`.
   - `limit=N` caps results without piping to `head`.

   `-ss` matches **short names** — `Module.Foo` queries match `Foo`. To find a method on a class, query the bare method name and read the qualified result. See `-ss help` for the full grammar.

4. **Avoid grepping through `arend-lib/src/` at this stage.** Bodies are large, full of metas, and not the part you need to read to decide if something exists. Use the retrieval flags instead — grep is the fallback when you can't phrase the question as a name pattern (`-ss`) or a signature shape (`-ps`).

5. **Decide: postpone, reject, or proceed.** If a prerequisite is missing, postpone or reject (and note what's needed) — don't start formalizing the goal on a foundation that doesn't exist yet. If the prerequisites are there, move to phase 2 with a list of signatures you intend to use.

---

## Phase 2 — While formalizing: structural lookup

Goal: pick the right classes, instances, and lemmas as you draft the statement.

1. **`arend -ch <class>`** — print the super/sub-class hierarchy of a class, with `\new` and `\instance` sites. Use when deciding whether to introduce a new class or extend an existing one, or when figuring out which class your statement should be parameterized over.

2. **`arend -fu <name>`** — find all usages of a definition. Accepts a bare short name (`arend -fu linarith`), a partial qualifier (`Monoid.pow`), or the full `MODULE:DEF` form; ambiguity is reported with the candidate list (`'pow' is ambiguous. Use one of: …`). Use when a definition's *purpose* isn't obvious from its signature (callers' pattern-of-use reveals the role); when checking whether something is used at all before refactoring (`No usages.` is the clear answer); or as a "does this fact exist under some weird name?" probe — `-fu` on a *constant* that MUST appear in the statement of the lemma you're searching for (e.g. `BigSum`, `cabs`, `partialSum`) lists every site mentioning it, so a missing match rules out the lemma's existence.

3. **`arend -ps <pattern>` (proof-search)** — find lemmas by signature shape rather than name. The classic use is "I need commutativity of `+` but I don't know what it's called": `arend -ps "_ + _ = _ + _"`. Pass `-ps help` for the grammar. **Reach for it reflexively whenever you're about to write a stating-shape lemma** — a *negative* result ("No matches") is just as valuable as a hit, since it tells you the helper genuinely doesn't exist and saves you from a wasted lemma signature. Yes, it's slow (~20s on resolution); the information value is high.

These three combine: `-ss` finds candidates, `-ch` tells you the structural context, `-fu` shows you how to use them, `-ps` finds them when you only know the shape.

---

## Phase 3 — After writing the statement: get to "only GOAL errors remain"

Goal: a typechecking file where the only remaining errors are the `{?}`s representing unproven obligations. Stop here — proof work belongs to a later session and the **arend-prove** skill.

1. **Update `\import`s manually first.** Add the imports you remember needing. Don't aim for completeness — the next step will fill gaps.

2. **Typecheck.** The default workflow — no flag, just a positional target. Narrow the scope with the positional to keep the output readable:

   - `arend .` (no target) — the whole library.
   - `arend . <Module>` — that module and its cone.
   - `arend . <Module>:<def>` — one definition.

   With a daemon up (phase 0) these route to it automatically and return in seconds. Resolution and typechecking happen in the same pass, so unresolved references surface as `[ERROR]` alongside genuine type errors. Existing `.arc` caches load unless `-r` is passed; fresh ones are written back **only if `--serialize` was given** (on the `-d` bootstrap, for daemon-served calls). Reach for `-r` only when the binaries are out of sync, typically after a class refactor — and remember it needs `--no-daemon` to have any effect, plus `--serialize` to leave a repaired cache behind.

3. **Fix unresolved references by hand.** For each `Cannot resolve reference 'X'`, find the owner with `arend -ss X` — the result is qualified, so it tells you both the name to write and the module to `\import`. When several modules define the same short name, `arend -sc <referable>` dumps the ambient scope at that position and shows which one is actually visible.

4. **Chase remaining typecheck errors.** Use `-ss`/`-ps`/`-fu`/`-ch` freely while triaging — these four cover most "what does this name mean / where else is this used / what's the right lemma" questions without leaving the terminal.

5. **Capture knowledge as you go.**
   - If you learned a *language-level* fact (a quirk, a syntax surprise, a typing rule that bit you), add it to `~/.claude/skills/arend-quirks/SKILL.md`.
   - If you learned a *meta-level* fact (a failure mode, a non-obvious meta interaction), add it to `~/.claude/skills/arend-prove/SKILL.md`.
   - If something was **cumbersome, slow, or painful** in a way that feels like a *tooling* issue rather than your mistake (typechecker quirk, slow/misleading CLI behavior, confusing diagnostic), append it to `arend-issues.md` at the repo root — `/home/sergey/Documents/Arend/arend-issues.md`. The file was deleted on 2026-05-21 (`f0c5f9dbf`) and **recreated on 2026-08-03** at the user's request to hold CLI/typechecker feature requests; its old numbered entries are gone, so never cite "`arend-issues.md` #N". **Do not** log missing library statements here — a missing lemma is not an issue, it's something to add. If a helper is missing from arend-lib, put it in the right module and move on.

6. **Done when the only remaining errors are `GOAL` (unproven `{?}`s).** Type-mismatch, unresolved-reference, universe-level, and termination errors must all be gone. If any non-GOAL error remains, you're not done.

7. **Re-read the final statement against the informal claim.** Variable order, implicit/explicit splits, universe levels, and class parameters all change meaning. The Arend statement that typechecks is not necessarily the formalization of what you were given — verify by hand.

---

## Phase 4 — the finish checklist

**Execute this as a pass, with the diff open, item by item. Do not run it from memory.** Every item below was violated in work that had this skill loaded, so recall is not the failure mode; execution is. Groups 5 and 7 are *experiments*: make the edit, re-typecheck, and put it back only if the typechecker objects.

*Provenance: the two "Refactor AI slop" reviews — `85817d37` (2026-06-16, 43 files, −3270/+2297) and `0e2fc701` (2026-07-31) — consist almost entirely of these eight groups.*

### 1. Goals and dead code
- `grep -n '{?}'` over every file touched. A `{?}` must never reach a commit.
- `-fu` every new definition. `No usages.` → delete it.
- No lemma whose body is `idp`; no `pmap f idp` step; no lemma that is a partial application of another (`f a b 0`).

### 2. Naming — check every new name against this table

| Kind | Convention | Right | Wrong |
|---|---|---|---|
| `\Prop`/`\Type`-valued predicate or index family | Capitalised | `OnUnitCircle`, `DyadicShift`, `IsCentral` | `onUnitCircle`, `dyadic-shift` |
| "f commutes with g" | `f_g`, underscore | `fromReal_pow`, `conv_fromRat` | `fromReal-pow`, `fromRat-conv` |
| adjectival / qualified variant | `f-adj`, hyphen | `abs-square`, `pow_<=-degree` | — |
| coefficient | `coef` | `compose-coef`, `coef-Rat` | `coeff` |
| one of a directional pair | direction in the name | `exp_negative-left` / `-right` | `exp_negative` |
| abbreviation | spell it out | `square` | `sq` |

Also: no proof-plan labels in names or doc comments (`L1:`, `Step 3:`, `Main theorem:`); no file-local `\alias` for a function used fewer than ~10 times — use `\open M \using (Long \as short)`.

### 3. Comments — delete on sight
- Any comment that restates the signature in words.
- Any comment documenting *your* workaround ("stated over an array variable so the implicits infer"; "naming it keeps `pmap` from guessing").
- Any proof sketch duplicating the body, and any `-- Step N` marker inside a body.
- Any comment narrating the file organisation ("supporting material lives with its subject matter").
- Any `TODO: hoist to X` / `TODO: belongs in \where of X` — do the move now, it is two lines.

Then: every surviving comment that asserts a mathematical necessity ("needs commutativity", "not cancellative at ∞") must be one you can defend. Two of mine were false.

### 4. Placement
- Exactly one consumer (`-fu`) → move into that consumer's `\where`.
- Takes `{X : SomeClass}` and is only ever used at `X` → make it a member of the class.
- `\where`-nested and needed by a sibling → drop `\private` or hoist to file level (**arend-quirks** §18).
- Directory matches subject. A real-analysis statement does not live under `Algebra/Field/`.
- No import from a higher layer: nothing under `Algebra/` imports `Analysis/` or `Topology/`. If you needed a type from there, define the algebraic one locally.
- Delete every import no longer used.

### 5. Hypotheses — try to weaken every one
The check nobody performs by reflex, and the source of the mathematically wrong signatures.
- Each explicit hypothesis: delete it, re-typecheck.
- Each class parameter: replace by each superclass (`-ch <class>` lists them), re-typecheck. Real instances: `CompleteExNormedRing` → `ExPseudoNormedRing` (7 lemmas), `OrderedCRing` → `LinearlyOrderedSemiring`, `BanachAlgebra` → `QAlgebra`.
- `Fin n` in a statement → try `{A : FinSet}`. Stated at `Fin n` it will not apply to `PairsFinSet` / `SigmaFin` / `ProdFin`, and you will re-prove it by hand.
- `\Pi (y z : A) -> y * z = z * y` is almost always too strong: the statement usually needs one element central (`Monoid.IsCentral`).
- Any argument that is a proposition and gets `prop-pi`'d downstream → mark `\property`.

### 6. Abstraction by repetition
- A hypothesis appearing in ≥3 signatures of this diff → name it.
- A class combination appearing in ≥3 signatures → declare the class. (`\class RealBanachCAlgebra \extends RealBanachAlgebra, CRing` is one line and deleted ten `*-comm` arguments.)
- Two definitions differing only in a constant → generalise one, delete the other.

### 7. Redundancy — omit and re-typecheck
- Every explicit implicit `{…}`.
- Every type ascription `(meta : T)` — let the chain position pin the type.
- Every `\lam (x : T) =>` binder annotation.
- `\new Array A n (\lam i => e)` → `\lam i => e`.
- **Qualification: `C.f {inst} args` → `inst.f args`.** Qualify with the *instance*, never with the defining class carrying the instance as an implicit. `RealField.pow x n`, not `pow {RealField} x n`. This one alone accounts for hundreds of tokens per file.
- `\lam x => f x + e` → `` (`+ e) ``; `transport (\lam x => P x) (inv p)` → `transportInv P p`.
- Hand-stepped `\have` chains where `simplify` / `equation.cRing` / `linarith` closes the goal outright.
- Hand-written `\case d \as d \return (\case d \with {…}) = …` motives → `mcases \with {…}`.
- `\case e \with { | inP (…) => body }` → `\let | (inP (…)) => e \in body`.

### 8. Layout
- Signature: parameters, then `: Result` on its own line, then `=> body`. Never `… (args) :` at end of line.
- `\where {` on its own line, members indented one level.
- `\have` for proofs, `\let` for values.
- `Record R`, not `Record { | R => R }`; `| Parent => …` to reuse a parent instance rather than re-implementing its fields.

---

## Common diagnosis recipes

### "The lemma is `[GOAL]` but I didn't write `{?}` in the body"

Look upstream, not at the reported def. A `[GOAL]` on a body you wrote in full usually means an *earlier* definition failed to typecheck, and the reference to it degraded into a hole. Re-run scoped to the whole module (`arend . <Module>`) rather than the one definition, and read the *first* `[ERROR]` in the output — that's the real bug; everything after it is fallout. Confirm by checking that def alone with `arend . <Module>:<upstream-name>`.

### "Library compiles cleanly per-module but the full-library refresh throws `IllegalArgumentException`"

Usually `linarith` / `equation` choking on a goal where the implicit algebraic class can't be inferred. Per-module checks may skip the failing path because of caching. The exception isn't attached to a source line, so you have to bisect by commenting out tactic calls. **Workaround**: replace `linarith` with an explicit lemma chain (e.g. `<_+-right`, `transport`, `zro-right`).

### "Module typechecks cleanly but downstream modules report bogus type mismatches like `Rat` vs `Arith.Rat.Rat`"

Stale binary cache after a class refactor. Only a forced recompile fixes it: `arend --no-daemon -r --serialize <module>`. All three flags are needed — a daemon-served `-r` is silently ignored, so without `--no-daemon` you get a warm run that reads the same stale state; and without `--serialize` the clean build writes no `.arc` at all, so the bad one survives and the next invocation is wrong again.

### "The daemon insists a definition I can see doesn't resolve"

Daemon poisoning: a transient resolution error (a duplicate name, a half-saved file) can leave the warm server denying a definition that is plainly present. `-r` won't help — it's one of the flags the daemon eats. Restart it: `arend . --daemon-stop && arend . -d`.

### "Cannot infer implicit X" inside a `\where`-block lemma calling its outer lemma

`\where`-private lemmas don't inherit the outer's implicit binders even when they syntactically reference outer variables. When recursing — or calling sibling private lemmas — pass the outer implicits explicitly: `outer-lemma {A} {b} {x} N` instead of `outer-lemma N`. Alternative: avoid `\where` and put helpers at top-level (still `\private`).

---

## Quick reference — which CLI flag for which question

| Question | Flag |
|---|---|
| "Does a lemma of this *shape* exist?" | `-ps` |
| "Is there a definition whose name matches X?" | `-ss` |
| "Where is this definition used?" | `-fu` |
| "What classes extend / instances exist for C?" | `-ch` |
| "Why isn't X in scope here?" | `-sc` |
| "I edited; resolve + typecheck." | no flag — `arend . <Module>[:<def>]` is the default workflow |
| "Make the edit loop fast." | `arend . -d` once per session; every later call auto-routes to the warm daemon |
| "Is a daemon up, and is it busy?" | `--daemon-ping` (`IDLE` / `BUSY`) |
| "I edited sources; refresh the daemon's view." | `--daemon-refresh` (re-runs `-ai` on the warm context) |
| "Resolution candidates + `.sig` mirror + index refresh." | `-ai [MODULE[:DEF]]` |
| "Ignore stale binary caches after a refactor or a hang." | `--no-daemon -r` (plain `-r` is ignored by the daemon) |
| "The daemon is wedged / denies a visible definition." | `--daemon-stop`, then `-d` again |

### `-ss` dialect reference

Plain mode is **literal substring on short names** — not regex (Arend identifiers use `*`, `^`, `$`, `?`, `|` freely so these stay literal). Prefixes select other modes:

| Prefix | Meaning |
|---|---|
| `re:<java-regex>` | Raw regex |
| `glob:<pat>` | Wildcards (`*`, `?`) |
| `glob:<text>` (no `*`/`?`) | Exact full *short* name — the old `eq:` mode was removed and now errors |
| `lit:<text>` | Forced literal (bypasses the regex-look check — use when a real Arend name looks regex-y) |
| `hb:<chars>` | Humpback / camel-boundary fuzzy |

Multi-pattern OR: whitespace inside one `-ss` argument splits into OR'd patterns — `-ss "A B C"` ≡ `-ss A -ss B -ss C`. Multiple `-ss` flags also OR. Filters (extra positional tokens, AND-combined): `contains=<text>` (repeatable substring AND), `kind=func,lemma,...`, `only=lib1,lib2|self`, `case-sensitive`, `limit=N`, `no-cache`.

Safety nets: a plain pattern containing a regex *sequence* (`.*`, `.+`, `.?`, `(?...`) is rejected with a fix-it; `|` emits a soft warning; a leading apostrophe in an unquoted arg warns about shell-bleed; each run echoes the parsed query so misparses are visible.

### Other inspection tools — when they pay off

- **`-ps <pattern>` — proof search by *signature shape*.** Slow (~20s on resolution) but unique: "is there a lemma whose parameters/codomain match this shape?". A negative result is as valuable as a hit — it tells you the helper genuinely doesn't exist. Default discipline: when about to write `\lemma foo : <shape> => {?}`, run `-ps "<shape>"` first.
- **`-fu <name>` — every usage of a definition.** Bare short name works (`arend -fu linarith`); ambiguity prints the candidate list. Read usages when the *purpose* of a def isn't obvious from its signature; check `No usages.` when refactoring; use it on a load-bearing constant (e.g. `BigSum`, `partialSum`) to verify whether a lemma you suspect to exist could exist under a different name.
- **`-ch <class>` — class hierarchy.** Super/sub-class trees plus `\new` and `\instance` sites. Use when deciding whether to introduce a new class or extend an existing one.
- **`-sc <referable>` — scope dump.** Lists everything in scope at a referable's position. Right tool for "why isn't X visible here?". Accepts an optional `-ss`-style pattern to filter.

---

## Style and refactoring

The compaction patterns now live in **Phase 4** above and are meant to be executed there, not recalled. Two that don't fit a checklist item:

- hoist repeated explicit embeddings to `\use \coerce` (see **arend-quirks** §10);
- prefer `-- |` doc comments over `{- | … -}` for one-liners (see **arend-quirks** §19 for why LaTeX in block comments is hazardous).

Treat all of it as the target style *while writing*; Phase 4 is the backstop, not the plan.

(There was once a separate `arend-refactor` skill holding worked examples. Its `SKILL.md` had been overwritten with a copy of `arend-error-extraneous-input`, so it was removed on 2026-07-28.)

---

## Anti-patterns

- **Omitting `--serialize` and expecting warm caches.** It is opt-in: without it *no* `.arc` is written, so every run re-typechecks from source and every “forced recompile” fix evaporates. Bootstrap the daemon as `arend . -d --serialize`. (Conversely, do not reach for `--no-serialize` — no such flag.)
- **Running the whole edit loop without a daemon.** Every call then pays JVM startup plus a full library load. `arend . -d` once at the top of the session is the single largest speedup available.
- **Expecting `-r` to recompile while a daemon is live.** It is parsed and silently discarded. Write `--no-daemon -r`, or stop the daemon first — otherwise you'll conclude "the recompile didn't fix it" from a run that never recompiled.
- **Reading `arend-lib/src/*.ard` to find a lemma.** Use `-ss` / `-ps` / `-fu` / `-ch` instead; open sources only once they've told you which module to open.
- **Writing the statement first, then deciding which class to parameterize over.** That decision should come out of `-ch` in phase 2, not from staring at the goal.
- **Chasing type errors while unresolved references remain.** Resolution and typechecking share one pass, so an unresolved name poisons every type downstream of it. Clear all `Cannot resolve reference` errors first, then re-read what's left — much of it will be gone.
- **Treating a typechecking statement as "done" without re-reading it against the informal claim.** Step 7 of phase 3 is non-negotiable — typechecking proves consistency, not faithfulness.
- **Skipping Phase 4 because the code typechecks.** Typechecking is the entry condition for Phase 4, not a substitute for it. The two review commits that motivated the checklist deleted ~1000 net lines of typechecking code.
- **Re-reading the Phase 4 list instead of running it.** Several of its items were already written down elsewhere in this skill set when they were violated. The list is only worth anything as an executed pass with a re-typecheck after each group.
