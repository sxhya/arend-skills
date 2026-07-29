---
name: arend-error-extraneous-input
description: Diagnose Arend parser errors of the form `extraneous input '<tok>' expecting {...}`. Use when typechecking an .ard file surfaces this exact error class; covers known surface-syntax constructs that trigger it and the minimal fix for each. The token reported is almost never the bug — the real offender is a few tokens earlier.
user-invocable: true
---

# Arend parser error — `extraneous input '<tok>' expecting {...}`

When Arend's parser gives up on an `.ard` file mid-expression, it prints something like:

    extraneous input '(' expecting {<EOF>, ...}

The reported token is **where the parser bailed**, not where the syntactic mistake is. The real offender is usually a clause, binding, or keyword a few tokens earlier — once the parser is desynchronized it will flag the next opening bracket / `\with` / similar.

Workflow when this fires:

1. Note the token and look at the **preceding** construct, not the token itself.
2. Match against the catalogue below. If a pattern fits, apply its fix.
3. If nothing matches, narrow by deleting the suspect clause/binding to see whether the error moves — that pinpoints which clause is malformed.
4. Once the root cause is identified, append it to this catalogue as a new subsection (date + minimal repro + fix + why).

---

## Known triggers

### `\have | : T => v` — anonymous `\have` clause (2026-05-08)

**Repro:** Inside `partialSum-uconv-lower` in `Analysis/CauchyProduct.ard` (that lemma has since been renamed — the nearest current definition is `partialSum-uconv-square`, `Analysis/CauchyProduct.ard:66` — but the syntax rule below is unchanged):

    \case ... \with {
      | ... => \have | : k < M => fin_< k \in linarith
                  ...
    }

**Error:** `extraneous input '(' expecting {<EOF>, ...}` — pointing at the opening paren of the `\with`/next construct, far from the actual problem.

**Fix:** Name the binding, even when the name is unused in the body:

    \have | hk : k < M => fin_< k \in linarith

**Why:** `\have | : T => v` is not valid surface syntax. The `|`-clause form of `\have` requires a name before the `:`; only the no-`|` form `\have x : T => v` exists and even there the name is mandatory. After failing on the clause, the parser misreports the failure at the next opening bracket.

---

---

### Identifier starting with a digit (2026-07-28)

**Repro:** naming a `\have` clause after the fact it proves, when the fact starts with a numeral:

    \have | 1+re>0 : 1 + u.re > 0 => linarith
          ...
     \in ... RealField.pinv (1 + u.re) 1+re>0 ...

**Error:** `extraneous input '+re>0' expecting {'=>', ':'}` — reported at the *binding site*, and the quoted token is the identifier with its leading digit shorn off.

**Fix:** start the name with a letter. arend-lib's convention is to lead with the subject: `den>0`, `re>0`, `a0>eps`, `r0>0`, `one/3>0`.

**Why:** Arend identifiers may contain `+ - < > / = *` freely but may not *begin* with a digit, so `1+re>0` lexes as the numeral `1` followed by the identifier `+re>0`. The parser then sees two tokens where it wanted one and blames the second. Note the fix is at the *definition*, even when the error is reported at a *use* site (or vice versa) — the same bad token appears at both.

---

## How to extend this skill

When you fix a fresh `extraneous input` instance whose root cause isn't in the catalogue above:

- Add a new `### <short tag> — <date>` subsection to **Known triggers**.
- Include a minimal repro, the misleading error, the fix, and a one-line "why" if known.
- Keep entries dated so stale ones can be culled when Arend's parser changes.
