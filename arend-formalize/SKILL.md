---
name: arend-formalize
description: Procedural workflow for turning an informal mathematical statement into a typechecking Arend definition/lemma. Use when the user gives an informal claim and asks for a formalization, or when starting a new .ard file from a math text. Covers what to read first, which CLI tools to reach for, and the order of operations from "I have an idea" to "only GOAL errors remain".
user-invocable: true
---

# Arend formalization workflow

A staged process for going from an informal claim to a typechecking Arend statement. The three phases are interleaved in practice — but resist the temptation to start writing `.ard` before phase 1 is done.

For language-level surprises while writing the statement, see the **arend-quirks** skill. For closing equality/algebra goals once the statement compiles, see the **arend-prove** skill.

---

## Phase 0 — Turn on `--serialize`

**Serialization is opt-in** (commit `9a59b3151`, 2026-05-15). Without `--serialize`, typechecking runs but **no `.arc` files are written** — so every invocation re-typechecks arend-lib from source (~2 min cold), and nothing you do makes the next run any faster. Pass it on every typechecking invocation:

```bash
cd /home/sergey/Documents/Arend/arend-lib && arend --serialize <Module>
```

The first run pays the full cost and persists the caches; subsequent runs deserialize the unchanged cone and only re-typecheck what you edited — seconds instead of minutes. That difference is the whole edit loop, so make `--serialize` the default on every typecheck, not something you remember occasionally.

Read and write are separate switches:

| | Behaviour |
|---|---|
| **Loading** existing `.arc` | On by default. No flag needed. `-r` / `--recompile` turns it *off* for one run. |
| **Writing** `.arc` after typecheck | Off by default. `--serialize` turns it *on*. |

**Drop `--serialize` when the cache itself is the suspect.** Persisted caches are the mechanism behind the phantom-error failure mode: after a class-hierarchy refactor, cached expressions referencing the old internal terms still deserialize cleanly and report errors that don't exist in the sources. Symptoms:

- Bogus type mismatches naming what looks like the same type twice (`Rat` vs `Arith.Rat.Rat`).
- Errors on definitions you never touched, which vanish under `-r`.
- A Java stack trace out of the (de)serializer rather than a typecheck diagnostic.

Recover with **`arend -r --serialize <Module>`**, not bare `-r`. `-r` only skips *loading* the caches (`BinaryLoader.setRecompile(true)`); it does not delete the `.arc` files on disk. So `-r` alone gives you one clean run while leaving the poisoned caches in place for the next plain invocation to load again. Pairing it with `--serialize` overwrites them from the clean build. Once the tree is green, stay on `--serialize` — don't leave it off permanently to dodge a cache bug, or you forfeit the entire speedup.

**Run every call from inside the library directory.** `cd arend-lib` first: the typecheck workflows and the retrieval flags (`-ss`, `-ps`, `-fu`, `-ch`, `-sc`) all expect the library in scope, and invoking them from the parent dir crashes with `NullPointerException: ctx.outputRouter is null` instead of a clean "no library in scope" error.

---

## Phase 1 — Before formalizing: orient yourself in arend-lib

Goal: figure out whether the prerequisites already exist, whether someone has already done this, and what names/classes to reuse. Cheap reads first, expensive reads last.

1. **Start with the retrieval flags, not the sources.** `-ss` / `-ps` / `-fu` / `-ch` / `-sc` query the loaded library without typechecking it, and they answer "does this exist / what shape is it / who uses it" far more cheaply than reading `.ard` files. Work out *which module* owns the concept from their qualified results, and only then open sources.

2. **When you do open a source file, read the signature, not the body.** Definitions in arend-lib are signature-first: the `\func` / `\lemma` / `\class` line plus its result type is what you need to decide whether something applies. Bodies are large, meta-heavy, and rarely relevant to an existence question — skip past them.

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

   - `arend --serialize` (no target) — the whole library.
   - `arend --serialize <Module>` — that module and its cone.
   - `arend --serialize <Module>:<def>` — one definition.

   Resolution and typechecking happen in the same pass, so unresolved references surface as `[ERROR]` alongside genuine type errors. Existing `.arc` caches load unless `-r` is also passed; fresh caches are written back only if `--serialize` is passed (see phase 0 — keep it on). Reach for `-r` only when the binaries are out of sync, typically after a class refactor, and pair it with `--serialize` so the poisoned caches get replaced rather than merely skipped.

3. **Fix unresolved references by hand.** For each `Cannot resolve reference 'X'`, find the owner with `arend -ss X` — the result is qualified, so it tells you both the name to write and the module to `\import`. When several modules define the same short name, `arend -sc <referable>` dumps the ambient scope at that position and shows which one is actually visible.

4. **Chase remaining typecheck errors.** Use `-ss`/`-ps`/`-fu`/`-ch` freely while triaging — these four cover most "what does this name mean / where else is this used / what's the right lemma" questions without leaving the terminal.

5. **Capture knowledge as you go.**
   - If you learned a *language-level* fact (a quirk, a syntax surprise, a typing rule that bit you), add it to `~/.claude/skills/arend-quirks/SKILL.md`.
   - If you learned a *meta-level* fact (a failure mode, a non-obvious meta interaction), add it to `~/.claude/skills/arend-prove/SKILL.md`.
   - If something was **cumbersome, slow, or painful** in a way that feels like a *tooling* issue rather than your mistake (typechecker quirk, slow/misleading CLI behavior, confusing diagnostic), append it to `arend-issues.md` at the repo root — `/home/sergey/Documents/Arend/arend-issues.md`. Note that this file **does not currently exist**: it was deleted on 2026-05-21 in commit `f0c5f9dbf` "Remove issues/blueprint" (699 lines, alongside `cos-pi-blueprint.md`), so its old numbered entries are gone and nothing should cite them. Recreate it only if the user wants the log resumed — ask before reintroducing a file they deliberately removed. **Do not** log missing library statements here — a missing lemma is not an issue, it's something to add. If a helper is missing from arend-lib, put it in the right module and move on.

6. **Done when the only remaining errors are `GOAL` (unproven `{?}`s).** Type-mismatch, unresolved-reference, universe-level, and termination errors must all be gone. If any non-GOAL error remains, you're not done.

7. **Re-read the final statement against the informal claim.** Variable order, implicit/explicit splits, universe levels, and class parameters all change meaning. The Arend statement that typechecks is not necessarily the formalization of what you were given — verify by hand.

---

## Common diagnosis recipes

### "The lemma is `[GOAL]` but I didn't write `{?}` in the body"

Look upstream, not at the reported def. A `[GOAL]` on a body you wrote in full usually means an *earlier* definition failed to typecheck, and the reference to it degraded into a hole. Re-run scoped to the whole module (`arend --serialize <Module>`) rather than the one definition, and read the *first* `[ERROR]` in the output — that's the real bug; everything after it is fallout. Confirm by checking that def alone with `arend --serialize <Module>:<upstream-name>`.

### "Library compiles cleanly per-module but the full-library refresh throws `IllegalArgumentException`"

Usually `linarith` / `equation` choking on a goal where the implicit algebraic class can't be inferred. Per-module checks may skip the failing path because of caching. The exception isn't attached to a source line, so you have to bisect by commenting out tactic calls. **Workaround**: replace `linarith` with an explicit lemma chain (e.g. `<_+-right`, `transport`, `zro-right`).

### "Module typechecks cleanly but downstream modules report bogus type mismatches like `Rat` vs `Arith.Rat.Rat`"

Stale binary cache after a class refactor. Only a forced recompile fixes it: `arend -r --serialize <module>`. Use both flags — `-r` ignores the `.arc` files but does not delete them, so without `--serialize` the stale caches survive on disk and the *next* plain invocation loads them right back. With the pair, the clean build overwrites them and subsequent invocations are both correct and fast. (See the user's `feedback_arc_cache.md` memory.)

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
| "I edited; resolve + typecheck." | no flag — `arend --serialize <Module>[:<def>]` is the default workflow |
| "Make the next typecheck fast (persist `.arc` caches)." | `--serialize` — pass it on every typecheck |
| "Ignore stale binary caches after a refactor or a hang." | `-r --serialize` (bare `-r` skips them without replacing them) |

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

Once your file reaches "only GOAL remaining", run a compaction pass before declaring it done — otherwise the proof will be rewritten in review. The patterns to apply:

- delete idp-bodied lemmas and `pmap f idp` steps;
- inline single-use helpers and single-use `\have` clauses;
- replace hand-stepped `\have` chains with `simplify` / `equation.cRing` / `linarith`;
- use slot notation (`` `<∘ bound ``, `__ + 1`) and `$` to flatten nesting;
- migrate helpers into `\where` where they belong to one parent;
- hoist repeated explicit embeddings to `\use \coerce` (see **arend-quirks** §10);
- prefer `-- |` doc comments over `{- | … -}` for one-liners (see **arend-quirks** §19 for why LaTeX in block comments is hazardous).

There was once a separate `arend-refactor` skill holding the worked examples for these. Its `SKILL.md` had been overwritten with a copy of `arend-error-extraneous-input`, so it was removed on 2026-07-28 and the list above is what survives. Treat these as the target style *while writing*, not only on a post-pass.

---

## Anti-patterns

- **Typechecking without `--serialize`.** Serialization is opt-in; omit the flag and no `.arc` files are written, so every single invocation re-typechecks arend-lib from source. Turning it off is for diagnosing a suspected cache bug, not a default.
- **Reading `arend-lib/src/*.ard` to find a lemma.** Use `-ss` / `-ps` / `-fu` / `-ch` instead; open sources only once they've told you which module to open.
- **Writing the statement first, then deciding which class to parameterize over.** That decision should come out of `-ch` in phase 2, not from staring at the goal.
- **Chasing type errors while unresolved references remain.** Resolution and typechecking share one pass, so an unresolved name poisons every type downstream of it. Clear all `Cannot resolve reference` errors first, then re-read what's left — much of it will be gone.
- **Treating a typechecking statement as "done" without re-reading it against the informal claim.** Step 7 of phase 3 is non-negotiable — typechecking proves consistency, not faithfulness.
