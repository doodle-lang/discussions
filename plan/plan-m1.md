# Working Plan: Milestone M1 — Front End

**Status:** Ratified by the user 2026-07-10 (as revised after review).
Blocked on M0.3 (core API), M0.4 (conformance runner + format), and M0.7
(insta/fuzz plumbing). · **Parent:** `implementation.md` §5 M1 `[M]`

Decomposition of M1 (lexer, parser, resolver, diagnostics) into
session-sized work items. Conventions as in `plan-m0.md`. Spec-delta
items (S-…) listed on a work item are **resolved in the spec as part of
that item** (edit + decision-log entry — L Appendix D or E Appendix B as
relevant — + conformance test citing the clause; plan §8); do not start
the code before pinning the spec text.

**Ratification note.** Once the user ratifies this plan, the S-item
resolutions *stated on the work items below* are pre-approved: a spec
edit implementing exactly the stated resolution needs no further
sign-off; anything beyond it — including any S-item marked "decide with
the user" — needs a fresh ask (CLAUDE.md: never change semantics without
asking).

**Conformance staging.** Work items land conformance tests at the stage
the pipeline supports when they close (`#! stage: lex` for M1.3–M1.5,
`stage: parse` for M1.6–M1.9 — see plan-m0's format v0); M1.10 upgrades
the suite to `full` where intended and is the checkpoint at which all
earlier conformance tests must be green at their final stage.

**M1 exit criteria (from the plan):** every code example in L parses to a
golden AST; a ~40-program broken-syntax corpus produces reviewed,
snapshot-tested diagnostics pointing at correct spans; every static-error
class has a position-asserting negative test; parser fuzzer survives 1 h.

---

## Work items

### M1.1 — Diagnostics infrastructure + error-message rubric — **DONE 2026-07-10**

`diag` module for real: `Diagnostic { severity, code, message, span,
notes, suggestion }`, multi-diagnostic `LoadError`, and a plain-text
renderer with source snippets + caret spans (the CLI/CI rendering; the
IDE renders its own from the structured form). Diagnostic `code` scheme
(pinned here): **provisional kebab-case slugs** (e.g.
`chained-comparison`, `const-reassignment`); a numbered scheme, if ever
wanted, is a future spec delta. Snapshot-test the renderer (insta).
**Write the error-message rubric** the plan references (§4.2) as
`discussions/plan/error-message-rubric.md`: names the value/operation,
points at the span, suggests the fix, kid-readable wording — with
examples of good/bad phrasings; every diagnostic-adding work item cites
it in review. The rubric itself needs **user sign-off no later than the
M1.13 review** (the agent must not both author the quality bar and
self-certify against it).
*Accept:* renderer snapshots for a hand-built diagnostic set;
rubric committed (user sign-off tracked for M1.13).
*Landed* (doodle-rust `9c49651`): the `diag` module + no-ANSI plain-text
renderer, 12 insta snapshots, CI green. `error-message-rubric.md` committed
and **signed off by the user 2026-07-10** (stays open to revision; exercised
at M1.13). Provisional D (structured `Replacement`) + F (code-point caret
columns; display-width → M1.2/S-1) **confirmed by the user**. Spec-deltas
filed (warnings channel, structured schema, ordering, CRLF→LF) — see
`../claude-todo.md`.

### M1.2 — Source model: NFC normalization, spans, positions (S-1) — **DONE 2026-07-10**

Source text NFC-normalized on load (L§3.1); `Span` = byte range into the
NFC'd text + derived line/col (1-based, code-point columns) per **S-1**
(resolve S-1's exact wording in L first: positions refer to
post-normalization text; positions/columns are in code-point units;
bindings may convert). Reconciliation note (so this doesn't read as a
conflict with Appendix C): the *internal* `Span` representation is byte
offsets; the *spec-facing* units are code points, derived at the
diagnostic/API boundary — that conversion is exactly what S-1's
"bindings may convert" licenses. UAX#31 identifier classification via
`unicode-ident` + `_`; module-name restriction (L§3.4).
*Accept:* unit + insta tests (identifier classification: é
composed/decomposed are one identifier; column derivation per S-1). The
corresponding conformance cases land with M1.3 (`stage: lex`), since no
`.doodle` file can be processed before the lexer exists.
*Landed* (doodle-rust `6aa0a7b`): S-1 pinned in L§3.1 + Appendix D.1 + E§8.1;
`src/unicode.rs` (NFC + UAX#31 identifiers + module names) and `src/source.rs`
(`Position`, `LineIndex`, `normalize`); `diag/render.rs` refactored onto the
promoted position API (snapshots byte-unchanged). Two follow-on spec questions
surfaced to the user (`claude-todo.md`): **XID vs ID** in L§3.4 (the code uses
NFC-closed XID; recommend the spec follow) and **CRLF→LF** normalization
(beyond S-1, deferred to a user decision).

### M1.3 — Lexer core: tokens, numbers, newline/continuation (S-2) — **TODO**

Token set (L§3.5–3.7); integer/float literals with underscore rules and
all bases; comments/shebang; **conditional NEWLINE emission** per the
continuation rules — resolve **S-2** first (pin the trigger set: symbolic
binary operators, comma, `and`/`or`/`is`; trailing `-`/`+` continue; `=`
does not; comments transparent) and land the L§3.2 edit with the tests.
*Accept:* token-dump snapshot tests incl. every continuation case in S-2;
negative tests (bad underscores, bad floats) with positions; conformance
`L3.2-*`/`L3.6.1-*`/`L3.6.2-*` tests green at `stage: lex` (plus the
M1.2 identifier cases).

### M1.4 — Lexer: strings, escapes, interpolation modes (S-47, S-48, S-49) — **TODO**

String literals with the full escape set; surrogate `\u{…}` as a static
error (L§3.6.3); the string↔expression **mode stack** for `{expr}`
interpolation (nested strings, nested braces, `{{`/`}}`); bytes literals
(ASCII-only source, `\u` rejected, no interpolation; L§3.6.5); NFC
applied to literal values. Resolve (stated resolutions in App C,
discussed with the user 2026-07-10): **S-47** — interpolations never
contain line terminators, in any string form (single-line stays
single-line; end-of-line error recovery preserved; no margin-stripping-
inside-expressions); **S-48** — empty `{}` (incl. whitespace-only) is a
lex error offering both intents (expression vs `{{`); **S-49** — the
escape set is closed (unknown escape = lex error; malformed known escape
= distinct diagnostic) and `\xHH` in a string denotes code point U+00HH
(byte in a bytes literal). Land the L§3.6.3/§3.6.4/§3.6.5/§6.7 edits +
App D.1 entries with the tests.
*Accept:* snapshot token dumps for nested-interpolation torture cases;
`L3.6.3-*`/`L3.6.5-*` conformance tests (`stage: lex`) incl. positions of
escape errors *inside* interpolations, the S-47 newline-in-interpolation
error (single-line and triple-quoted cases), S-48 empty-interpolation
error, and S-49 unknown/malformed-escape + `"\xE9" == "\u{E9}"` cases.

### M1.5 — Lexer: triple-quoted strings + margins (S-3) — **TODO**

Raw-span capture, closing-delimiter margin computation, per-line
validation, strip, then escape/interpolation processing (order per plan
§3.1). **S-3 is resolved (user, 2026-07-10 — full text in App C):**
exact-prefix margins (whitespace-string identity, no column arithmetic);
truly empty lines exempt → `""`, whitespace-only nonempty lines are
ordinary content lines (full margin + remainder-is-content); content
lines preserve trailing whitespace; no backslash-newline line-join;
nothing (incl. comments) after the opener, code may follow the closer;
mid-line `"""` is content, line-initial literal `"""` via `\"""`; value =
lines joined with `\n`, zero lines = `""`. Land the L§3.6.4 edit with
this item.
*Accept:* `L3.6.4-*` tests (`stage: lex`): margin stripping goldens
(incl. whitespace-only-line-contributes-whitespace and empty-line cases);
under-indented and **tab-vs-space mismatch** errors with position and the
first-differing-character diagnostic; escapes-don't-create-lines;
comment-after-opener error; `\"""` line-initial-literal case;
zero-content-lines = `""`.

### M1.6 — Parser: expressions (Pratt) — **TODO**

Primary/postfix/unary/binary per the 9-level table (L§6.5, App A/C);
right-assoc `**`; **non-associative comparisons** rejected with the
write-`a < b and b < c` guidance diagnostic; `if`/`try` expression forms;
anonymous `fn`; list/dict literals with trailing commas; keyword
arguments in call syntax (`name: expr`, positional-before-keyword).
*Accept:* golden AST dumps for an expression corpus; `L6.5-*` tests
(`stage: parse`) incl. the chained-comparison diagnostic with span.

### M1.7 — Parser: statements + construct headers (S-4) — **TODO**

Statement separation; `let`/`const`/assignment (lvalue grammar);
`if`/`while`/`loop`/`with` statements; `return`/`break`/`continue`/
`raise`; `try`/`rescue`. Resolve **S-4** first (header expressions parse
in no-trailing-block mode; block arg in a header requires parens) and
land the L§6.4/§7 edit.
*Accept:* `while f() do … end` parses as while-body (golden AST);
`while f() do … end do … end`-style ambiguities produce the S-4
diagnostic; statement-form conformance tests green at `stage: parse`.

### M1.8 — Parser: declarations + docstrings (S-27) — **TODO**

`to`/`fn` declarations (params, defaults, `do body` block param —
at most one, last); `record`/`ref record`; `protocol`/`extends` with
required-vs-default members; `implement … for …`; `module`; `parameter`;
`exports`; docstring capture as metadata. **S-27 is resolved (user,
2026-07-11 — full text in App C):** a lone string is the body's *result*
where the body produces a value (an `fn`'s return value — named or
anonymous), and the *docstring* otherwise (`to` and module bodies;
record/protocol bodies are docstring-only by grammar). First-string-
followed-by-more-statements is the docstring in every body kind.
Rawness follows classification: docstrings are raw (interpolation
inert); an `fn`'s lone-string result is evaluated normally. Land the
L§8.6 edit with this item.
*Accept:* golden ASTs for every declaration form in L's examples;
`L8.6-*` docstring tests (`stage: parse`) incl. the lone-string cases —
`fn` lone string parses as result (interpolation active), `to`/module
lone string as docstring (raw), doc+result `fn` gets both; placement-rule
negatives (module-level-only declarations, L§7.1) with positions.

### M1.9 — Parser: imports + calls + blocks — **TODO**

Import forms (`m`, `m as x`, `m.item`, `m.item as x`, `m.*`,
comma-separated; `.*` not renamable) with the **S-7 grammar note**
(module-vs-member is *resolved at load*, not parse — the parser produces
the dotted path; resolve S-7's resolution-order text in L§11.2 here,
implementation follows at M6); call argument lists; trailing block
arguments `do (params) … end`.
*Accept:* golden ASTs for L§11.2's example list; `L11.2-*` tests
(`stage: parse`; `.* as` rejected, keyword-after-positional enforced
elsewhere).

### M1.10 — Resolver: scopes, slots, static-error battery (S-5, S-6) — **TODO**

Scope construction (module/body/block, per-invocation scopes are a
runtime matter); slot assignment; capture marking (cell-boxed, per
machine-design §7 — **S-11 is resolved (user, 2026-07-17; full text in
App C)**: `fn` closures may mutate captures, capture-by-binding-
reference with sharing; land the L§6.10/§8.5 wording here); **static links**
(hop, slot) for block-body outer references (machine-design §7);
**free-name classification** — every unresolved name compiled to a
module-cell reference site (`name_refs`, machine-design §2/AD5), the
output M2a's machine consumes; **the static-error battery**: duplicate
declaration, `const` reassignment, the **full assignment-target rule**
(ruled 2026-07-17, see App C S-39: `name = v` is a static error unless
`name` resolves lexically to a mutable `let` — const, declarations (2a),
imports selective *or wildcard*, and undeclared names all error
uniformly, because S-39 makes every imported name read-only; no import
resolution needed), `return`/`break`/`continue` placement +
**lexical exit-target annotation** (machine-design §12),
fn-falls-off-end, chained comparison (from M1.6), surrogate escapes
(from M1.4), placement rules. (Only S-39's *wildcard provenance-naming*
diagnostic polish waits for M5.) **S-5 is resolved (user, 2026-07-16 — full text in App
C):** the tail-statement classifier with the four-way
produces/diverges/value-less/indeterminate lattice; loop divergence =
no *lexically bound* `break` (an exit-target-annotation lookup, not
reachability); condition-blind (dead-tail code rejected by design);
`fn` bodies only; tail-`while` error suggests `loop`. **S-6 is resolved
(user, 2026-07-16 — full text in App C):** consuming-site model with
producer-site blame; the full site enumeration (incl. `return`/`raise`/
`break`/`continue` operands and parameter defaults) under the
"every expression position consumes except the bare expression
statement" invariant; static where the S-5 lattice determines it,
runtime otherwise (runtime conformance at M2a); **plus companion rule
2a: declaration bindings are non-assignable** (a new battery member).
Land the L§6.11/§8.4/§8.5/§5.3 edits with this item.
*Accept:* every static-error class has ≥1 position-asserting conformance
test (`mode: static`, now `stage: full`; all earlier-stage tests
re-verified green at their final stages — this is the staging
checkpoint); the S-5 spec text enumerates exactly what M1 rejects (no
more, no less).

### M1.11 — Resolver: shadowing warning + tail marking — **DONE**

The L§5.1 shadowing **warning** (a Diagnostic with severity=warning; does
not fail loads; surfaced by runner/CLI); tail-position marking per L§8.7
+ machine-design §11 including the **S-45** exclusion (calls passing
block arguments are not tail — S-45's L§8.7 amendment landed here, ahead
of its M2a consumer).
*Accept:* warning snapshot tests; tail-marking unit tests over an
annotated corpus (marked tail sites match hand-annotations, incl.
with/try/block-arg barriers).
Tail marking lands as a per-node `tail_calls: Vec<bool>` on `ResolvedModule`
(`walk/tailmark.rs`); the walk also handles `return expr` operands (tail
wherever they sit) and treats `with`/`try`/block-arg/nested-callable
boundaries. Annotated-corpus tests use the `tails` helper in
`tests/resolve.rs`.

### M1.12 — Golden corpus: every example in L — **DONE**

Extract the code examples from `spec/language.md` into
`conformance/v0.1/lang/…` static tests (or, where an example is
deliberately ill-formed, negative tests); golden AST dumps (insta) for
the well-formed set. **Prerequisite spec edit (part of this item):** L's
~109 code fences are untagged and many are grammar/EBNF, not Doodle —
land a fence-tagging pass on language.md (```doodle vs ```grammar; also
improves rendering) or, failing that, a committed per-block manifest
(include/exclude/wrap-with-reason) that the extraction consumes;
fragmentary examples get a documented wrap or an exclusion reason.
*Accept:* the M1 exit criterion "every code example in L parses to a
golden AST" is mechanically true over the tagged/manifested set;
extraction script + manifest committed so spec edits flag stale examples.

Landed (doodle-rust `175de57`; tagging in discussions `30b20aa`): all 54
fences tagged (`doodle`/`grammar`/`text`); the 15 well-formed doodle
examples + #44 (§10.3 dispatch, `…` wrapped to `show(i)`) extracted to 16
`spec-bNN.doodle` fixtures (`mode: static`, `stage: full`) + one insta AST
snapshot each; `scripts/lang-corpus-sync.py` + `conformance/lang-corpus.json`
give the extraction+manifest with drift detection (both were done — tags
*and* a manifest — since the Accept line names the manifest unconditionally).
No deliberately-ill-formed L examples exist, so no negative tests. The
sync-check is not CI-wired yet (a future job pulling `discussions`).

### M1.13 — Broken-syntax corpus + message review — **agent pass DONE; user sign-off PENDING**

~40 hand-written broken programs (one mistake each, kid-plausible:
missing `end`, `=` vs `==`, unclosed string, stray `do`, tab-margin
mix…), snapshot-tested through the M1.1 renderer, **each message reviewed
against the rubric. Reviewer = the user** (an agent rubric-pass first
writes per-program notes into the corpus README; the user then approves —
this review, and the rubric sign-off from M1.1, are **blocking for
M1.15**).
*Accept:* the corpus README contains the filled review table with user
sign-off for all 40; each diagnostic points at the correct span.

Agent pass landed (doodle-rust `3066a7d`): 41 programs + `tests/broken_syntax.rs`
render-snapshots + the rubric-pass table in `tests/broken-syntax/README.md`
(22 PASS / 11 NEEDS-WORK / 8 FAIL), hardened by two read-only reviews. Scope set
with the user: catalog + fix-spans-only; message-quality fixes (cascade
suppression, `=`/`==` & comma suggestions, de-jargon) spun off to `claude-todo.md`.
**Remaining for M1.13:** the user's sign-off, then the in-scope span fixes (point
unclosed-construct errors at the opening token, not EOF — one systematic fix).

### M1.14 — Parser fuzz targets + soak — **TODO**

cargo-fuzz targets: lexer (arbitrary bytes) and parser (token-level or
text); a short fuzz job in CI (minutes); a 1 h local soak for the exit
criterion.
*Accept:* 1 h soak with zero panics/hangs/OOMs; CI fuzz-smoke job green.

### M1.15 — M1 exit review — **TODO**

Walk the plan's M1 acceptance; S-item audit (S-1…S-7, S-11, S-27, S-45
resolved in spec with tests); update `claude-todo.md` + status markers;
hygiene + CI green; file discovered deltas.

**Suggested order / parallelism:** M1.1 → M1.2 → M1.3 → {M1.4, M1.5} →
M1.6 → {M1.7, M1.8, M1.9} → M1.10 → M1.11 → {M1.12, M1.13, M1.14} →
M1.15. Braced groups are parallelizable across sessions **only if**
working on separate branches with the interfaces from earlier items
frozen; single-session-at-a-time is the default assumption.
