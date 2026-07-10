# Doodle error-message rubric

The quality bar every Doodle diagnostic — static (load-time) **and** runtime —
must meet. Every diagnostic-adding work item cites this rubric in review
(implementation plan §4.2; plan-m1 M1.1). The plain-text renderer that shows
diagnostics on the CLI is `doodle-core`'s `diag::render`; the structured form
(`diag::Diagnostic`) is what IDEs consume.

## Status

> **Signed off by the user (2026-07-10).** Drafted by the implementing agent
> and ratified by the user; it remains open to revision, and the M1.13
> broken-syntax message review is where it is exercised in earnest. Per plan-m1
> M1.1/M1.13 the agent must not self-certify — this bar governs the agent's
> rubric-pass, but the user approves the M1.13 messages.
>
> Signed off by: the user · Date: 2026-07-10

## 1. Purpose & scope

Doodle is a kid-first teaching language. A diagnostic is a *teaching moment*,
not a compiler complaint: the reader is often a beginner who made an ordinary
mistake and needs to understand **what** is wrong, **where**, and **how to fix
it** — in their own vocabulary. This rubric applies to every diagnostic the
engine emits, at every stage. M1 renders only static/load diagnostics, but the
bar is project-wide.

## 2. The four required elements

Every diagnostic must have (a) and (b); (c) and (d) apply whenever they can:

- **(a) Name the value or operation.** Say *which* thing is wrong using the
  program's own terms — the variable/procedure/function name, the literal, the
  operator — not an internal category. Maps to `Diagnostic::message`.
- **(b) Point at it.** Attach the primary `span` to the exact offending source,
  and use `notes` for a second relevant location ("the original is here").
  Maps to `Diagnostic::{span, notes}`.
- **(c) Suggest the fix.** When there is a clear correction, say it in words
  (and, when it is a mechanical edit, carry a machine-applicable
  `Suggestion::replacement` for the IDE). Maps to `Diagnostic::suggestion`.
- **(d) Be kid-readable.** Short, concrete, blame-free. Prefer a plain sentence
  over a term of art. A nine-year-old who reads it should know what to try
  next.

## 3. Voice & vocabulary

- **Use the language's own type names**, never Rust's: `Int`, `Float`,
  `Number`, `String`, `Bytes`, `Bool`, `Nil`, `List`, `Dict`, records,
  callables, modules, types, protocols. Never `i64`, `&str`, `Vec`, `usize`.
- **Honor the `to` / `fn` distinction** (L§1.3, §6.11): a `to` is a
  *procedure* (does something, yields no value); a `fn` is a *function*
  (produces a value). Name them that way in prose.
- **Second person, present tense, active voice.** "You can't reassign `pi`",
  not "reassignment of const identifier detected".
- **Avoid compiler jargon**: no "token", "lexeme", "lvalue", "rvalue", "AST",
  "non-associative", "Void", "symbol". Say the plain thing instead.
- **No blame, no exclamation marks, no cutesiness.** Calm and matter-of-fact.

## 4. Determinism

Diagnostics are on a determinism-gated path (workspace CLAUDE.md; engine spec
E§11). Any "did you mean …?" suggestion or candidate ranking must be
**deterministic**: rank by a fixed metric (e.g. edit distance) over a stable
order (source order / declaration order), with deterministic tie-breaking —
never an order derived from a default-hasher `HashMap` or from addresses.

## 5. Good / bad examples

| Situation (clause) | Good | Bad |
|---|---|---|
| Chained comparison (L§6.5) | `` comparison operators don't chain — write `a < b and b < c` instead `` | `non-associative operator '<'` / `unexpected token '<'` |
| Procedure used as a value (L§6.11) | `` `draw_house` is a procedure (a `to`), so it doesn't produce a value to use here — call it as its own statement instead `` | `type error: expected value, got Void` |
| Undeclared name / typo (L§5.3; NFC + case-sensitive, L§3.1/§3.4) | `` there's no variable named `coutner` here — did you mean `counter`? `` | `undefined symbol: coutner` |
| Reassigning a constant (L§5.2) | `` you can't reassign `pi` — it's a constant; use a new name, or declare it with `var` if it needs to change `` | `const-reassignment error` |
| Missing protocol (L§10.3; runtime) | `` a `Circle` can't be used with `each` — `Circle` doesn't implement `Iterable`; add `implement Iterable for Circle` with an `each` method `` | `no method 'each' for type Circle` |
| Shadowing (L§5.1; warning) | `` this `count` hides an outer `count` declared earlier — that's allowed, but check you meant to `` | `shadowing detected` |

## Appendix A — provisional diagnostic-code catalog

Codes are stable kebab-case slugs (`DiagnosticCode`); a numbered scheme, if
ever wanted, is a future spec delta. The reserved slugs below are added to the
enum only when their producer lands (M1.3–M1.11); each names an L rule.

`chained-comparison` (L§6.5) · `const-reassignment` (L§5.2) ·
`assign-to-undeclared` (L§5.3) · `duplicate-declaration` (L§5.2) ·
`return-outside-callable` · `break-outside-loop` · `continue-outside-block`
(L§7) · `function-missing-value` · `procedure-in-expression` (L§6.11) ·
`if-expression-missing-else` (L§6.8) · `misplaced-declaration` ·
`bare-raise` · `undeclared-parameter` · `malformed-number` · `bad-escape` ·
`surrogate-escape` · `unicode-escape-in-bytes` · `non-ascii-bytes` ·
`unterminated-string` · `under-indented-line` (L§3) ·
`keyword-as-identifier` · `reserved-word` · `invalid-identifier` ·
`invalid-module-name` · `wildcard-import-rename` ·
`positional-after-keyword` · `shadowing` (warning, L§5.1) ·
`discarded-value` (warning, L§6.11).

## Appendix B — provisional caret / column model

The M1.1 renderer reports line/column and caret widths in **code points** over
the NFC source (S-1-aligned). This is exact for most kid-authored source but
does not yet account for **display width** — tabs, East-Asian-wide characters,
and combining marks can misalign the caret in a terminal. Resolving the display
model is deferred to **M1.2/S-1** (which pins the canonical position API); the
`diag::render` `line_col`/`line_bounds` helpers are the seam that upgrade grafts
onto.
