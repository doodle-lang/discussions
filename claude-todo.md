# Doodle TODO / Status

The living work queue and status snapshot for the Doodle project — the
first thing a fresh session should read after the workspace `CLAUDE.md`.
Keep it current: update when work starts/lands, and record discovered
bugs per the Bug Discovery Protocol (CLAUDE.md). Detail lives in the
working plans (`plan/plan-m*.md`); this file is the index.

Conventions: `[ ]` todo · `[~]` in progress (note the session/branch) ·
`[x]` done (move to the Done log with date+commit). CRITICAL/MAJOR bugs
go at the top, per CLAUDE.md.

## CRITICAL

(none)

## MAJOR

(none)

## Awaiting the user (blocking)

(none — the M1.1 error-message rubric was signed off by the user 2026-07-10;
it stays open to revision and is exercised in earnest at the M1.13 review.

Context: plans ratified, machine-design v0.2 accepted, repos deliberately
public, issues enabled on discussions + doodle-rust: all 2026-07-10. The
S-27 semantic fork is decided with the user at M1.8 start — options +
recommendation in plan-m1 M1.8.)

**Non-blocking, for confirmation:** the M1.3 lexer spec edits — S-2
continuation triggers (L§3.2), numeric-literal lexing (L§3.6.1/§3.6.2), and
the M1.3-split rulings (`.`/`:` are not continuation triggers; plain-string
boundaries lexed now) — were made on the user's "go"/best-judgment while
away. All reversible; the lexer now builds on them, so flag for a look but
not blocking further work.

**S-50 RESOLVED (user, 2026-07-10): option (b) — DONE.** `#` inside any
string's `{…}` is a distinct lex-time error at the `#`'s span ("a comment
can't appear inside a string's `{…}`"; suggest moving the comment outside or
binding a named local); uniform across string forms. Implemented: the
targeted diagnostic + pinning test (doodle-rust `0a75678`) and the L§6.7
body rule (discussions `d96cc33`).

## In progress

- [~] **M1.6** — Parser (Pratt expressions). **M1.6a–b landed** (Done log):
  (a) the operator-precedence tower (L§6.5) + numeric lowering; (b) string/bytes
  literals — escape/value decoding + `{ … }` interpolation assembled from the
  lexer's structured stream, plus the line-final-`\` decode decision
  (provisional: a decode error; spec-delta queue). No stage bump. Remaining:
  - **M1.6c** — postfix `.`/`[]`/`()` with keyword args (`name: expr`,
    positional-before-keyword); list/dict literals with trailing commas.
  - **M1.6d** — `if`/`try` expression forms; anonymous `fn`.
  - **M1.6e** — stage bump to `Parse` + conformance-runner parse-path
    (run_parse) + `L6.5-*` `stage: parse` fixtures (atomic, like M1.3b).

## Next up

Milestone **M1 — Front End** (see `plan/plan-m1.md`): M1.1 … M1.15, with
conformance tests landing at `stage: lex/parse` per item and upgraded to
`full` at the M1.10 checkpoint. M1 was blocked on M0.3/M0.4/M0.7 — all done.
M1.1, M1.2, and M1.3 (lexer core + stage:lex conformance) have landed.

- [ ] **M1.4** — next: string literals proper (escape decoding, the grapheme
      model, triple-quoted strings, interpolation per L§3.6.3/§3.6.4/§6.7).
      The M1.3 `lex/string.rs` only scans plain-string boundaries (value-free);
      M1.4 does the decoding. See the M1.3 forward notes in the spec-delta
      queue, and **S-47/S-48/S-49** (filed 2026-07-10 with user-agreed
      resolutions — pairings + stated text in plan-m1 M1.4 / App C).

**Resolved at M1.3b (was an M0-exit heads-up):** the runner's stage gate is
now lifted — `implemented_through()` returns `Some(Stage::Lex)` and the
conformance runner's execute + expectation-matching path landed atomically
(doodle-rust `7210c3b`), so the old forcing-`Err` coupling is gone. Future
stage bumps (parse/full/run) must likewise co-land their executor arm in
`tools/conformance-runner/src/matcher.rs`.

**M2a gate:** satisfied — `plan/machine-design.md` v0.2 accepted by the
user 2026-07-10. (Mechanism changes still require revising that document
first.)

## Spec-delta queue (near-term)

Full backlog: `plan/implementation.md` Appendix C (S-1…S-49; the
appendix is the tracker of record until GitHub issues open). Due with M1
work items (resolve in spec *before/with* the implementing item —
pairings in `plan-m1.md`): S-1 (M1.2, done), S-2 (M1.3, done), **S-3
(M1.5 — margins resolved with the user 2026-07-10: exact-prefix; empty
lines exempt, whitespace-only nonempty lines are content lines; preserve
trailing whitespace; no line-join; no comment after opener — full text in
App C; M1.5 lands the L§3.6.4 edit)**, S-4 (M1.7), S-5 + S-6-in-full
(M1.10), S-7 (M1.9), S-11 (M1.10),
S-27 (M1.8, user decision), S-45 (M1.11), **S-47/S-48/S-49 (M1.4:
interpolations never contain line terminators in any string form; empty
`{}` is a lex error; closed escape set + `\xHH` = U+00HH in strings —
resolutions discussed with and agreed by the user 2026-07-10)**. Later:
S-41 by M2a; **S-9** (machine-design §12 carries the proposed
resolution) by M2a; **S-46** (non-local exits through native consumers)
by M2b.

Discovered at M0.3 (needs an S-number when the user next curates Appendix
C): **top-level `Completed` value** — E§7.2 pins the `Completed(value?)`
payload only for a returning `fn`; it does not say what a *top-level*
module drive completes with. The M0.3 skeleton provisionally returns the
result register's last value so the acceptance can observe it; real
top-level completion is expected to be Void (`None`). Resolve in E§7.2 by
M2a (when the machine core replaces the placeholder). No behavior ships
before then.

Discovered at M1.1 (diagnostics; resolve in E Appendix B / L Appendix D at
the milestone that ships the behavior — provisional choices documented in
code, nothing shipped that contradicts a future spec pin):
- **Warnings channel at the load boundary** — E§3.2 `load -> Module |
  LoadError` has no "loaded OK + N warnings" outcome, yet L§5.1 warnings
  occur on a *successful* load. The `diag` types + renderer already handle a
  bare `&[Diagnostic]`; pin the boundary shape in E by the milestone `load`
  lands.
- **Structured diagnostic schema** — E says only "LoadError carrying
  positions"; the IDE consumes the structured `Diagnostic`
  (severity/code/message/module/span/notes/suggestion). Pin the schema in E
  when a binding/IDE consumer ships.
- **Diagnostic ordering** — multi-diagnostic order is a producer contract
  (nondecreasing `span.start`, tie-break production order); the renderer
  never re-sorts. Pin in E Appendix B.
- **Line endings (CRLF→LF)** — **RESOLVED (user, 2026-07-10): normalize
  CRLF→LF at load.** A CRLF (CR immediately before LF, §3.2) is replaced by a
  single LF before NFC, so the source model, spans, columns, and the lexer see
  LF-only text (a lone CR is left as is). Landed: `source::normalize` +
  L§3.1/Appendix D.1 (doodle-rust `8826655`).

Discovered at M1.3 (lexer; resolve in L / L Appendix D at the milestone that
ships the behavior — provisional choices documented in code, nothing shipped
that contradicts a future pin):
- **Inline-whitespace set** — L§3.2 speaks of "whitespace" between tokens but
  never enumerates it. The lexer treats **space (U+0020) and tab (U+0009)** as
  inline whitespace (plus a lone CR, below); no other Unicode space separators
  are recognized (a kid-first language wants exactly one invisible indent
  character story, and NBSP-as-space is a classic footgun). Pin the set in L§3.2
  when the lexer's whitespace behavior is spec-fixed.
- **Lone CR** — the CRLF→LF resolution (above) leaves a **lone** CR (a CR not
  immediately before LF) in the source. The lexer's `skip_inline` then treats
  that lone CR as inline whitespace (so an old Mac `\r`-terminated line does not
  wedge the lexer). This is a lexer choice not yet stated in L; pin it alongside
  the inline-whitespace set. (Columns still count the CR as one code point, per
  the §3.1 position model.)
- **Forward notes (not spec-deltas), for M1.4 string/grapheme work:** (a)
  `lex/string.rs` currently scans string *bytes* only far enough to find the
  closing quote (backslash-escaped quotes don't close) — escape **decoding**,
  the grapheme model, and interpolation are M1.4+, and the scanner is
  deliberately value-free. (b) `bracket_depth` is a single un-matched counter
  (continuation only); real bracket **matching** and mismatch diagnostics are
  the parser's job (M1.6). Both are noted in code.

Discovered at M1.5, provisionally resolved at M1.6b (user to confirm + pin):
- **Line-final backslash in a triple-quoted string** — a `\` at the end of a
  content line (before the `\n` line break). **Provisional (M1.6b): a decode-time
  error** ("a backslash here isn't a valid escape"), consistent with the closed
  escape set (L§3.6.3: "a backslash followed by anything else is a static error")
  and S-3 rule 4 (no backslash-newline continuation). Implemented in the parser's
  string decode + tested. **User to confirm and pin the sentence in
  L§3.6.3/§3.6.4.** (A single-line unterminated string ending in `\` gets both
  unterminated-string and this — minor, on already-errored input.)

Discovered at M1.6b (parser string decode; non-blocking):
- **NFC normalization of string-literal values** — L§3.6.3/§4.4 make a String's
  value NFC-normalized. The parser stores the decoded literal text in the AST
  *un-normalized* (`"e\u{301}"` → 2 code points, not composed `é`). This is fine
  at the AST layer — normalization is specified to happen at String
  *construction*, which the evaluator (M2) does (and re-does on concatenation) —
  but the literal→`String` construction must apply NFC (including to a pure
  single-text-part literal). Flagged so it isn't forgotten when eval lands.

Resolved at M1.2 (user decisions, 2026-07-10 — language-semantics changes):
- **Identifiers: `XID` not `ID` (L§3.4)** — **RESOLVED (user): change L§3.4 to
  `XID_Start`/`XID_Continue`.** The NFC-closed variants (matching the code /
  `unicode-ident` / UAX#31 default), correct for an NFC-normalizing language.
  Landed: L§3.4 + Appendix D.1 + AD4 CI-vector text; code confirmed; a U+037A
  test pins it (doodle-rust `8826655`).

## Open decisions awaiting the user (non-blocking)

From `plan/implementation.md` §10: D-2 (spec home at freeze), D-4 (names/
npm scope), D-5 (Unicode pin verification), D-6 (demo posture), D-7
(privacy/analytics), D-8 (hosting + release cadence). D-1 and D-3 are
resolved (but see the visibility discrepancy above).

## Done

- 2026-07-11 — **M1.6b: string/bytes literal assembly + escape decoding.** New
  `parse/decode.rs` (closed escape set; `\xHH`=U+00HH in strings / byte in bytes;
  `\u{…}`; `{{`/`}}` → `{`/`}`; panic-free best-effort on lexer-errored input) +
  `string_lit` assembling `StrStart/StrText/interp/StrEnd` into a `StrLit` (merged
  text parts, parsed interpolations, nested strings) and `BytesLit`. Line-final-`\`
  → a decode error (provisional S-resolution). Read-only review found a CRITICAL
  char-boundary panic on a malformed `\x` before a multibyte char (`"\x1é"`) —
  fixed (advance only past real hex digits) + regression + ~13.8k-combo brute
  stress, no panic; two minors folded in (bytes suffix-strip; interp double-error).
  doodle-rust `bef25ab`.
- 2026-07-11 — **M1.6a: Pratt expression parser core** (L§6.5 tower). New
  `parse.rs` (precedence climbing; binding powers per the 9-level table;
  right-assoc `**`; non-assoc comparison → chained-comparison, tracked per
  climb level so `(a==b)==c` is not misflagged; MAX_DEPTH guard vs stack
  overflow) + numeric value lowering (i64/bignum/f64). `ast.rs` grown with the
  expression `Node`/`UnaryOp`/`BinaryOp` (machine-design §2 flat arena).
  AST-dump tests make precedence/associativity/lowering visible. Read-only
  review found 2 majors (parenthesized-comparison misflag; unbounded recursion)
  — both fixed + regression-tested. No stage bump. doodle-rust `da9f830`.
- 2026-07-11 — **M1.5: triple-quoted strings + S-3 margins** (code). Spec gate
  `b7bbebb` (S-3 into L§3.6.4). Code (doodle-rust commit below): `lex/string.rs`
  `scan_triple_string` — two-pass (find the closing `"""` + its margin, then
  strip byte-for-byte per line and emit content as `StrText` with inter-line
  `\n`-join chunks; empty-line exemption; nothing-after-open + margin-mismatch
  diagnostics); reuses the M1.4 stream/interp machinery via a `triple` flag on
  `scan_text_run`. 3 `L3.6.4` fixtures; suite 17/0/3. A double-advance bug
  (extra `pos += 1` after an `emit`) was caught by the value tests and fixed;
  two multibyte-span nits fixed. 2-lens read-only review: both SHIP (margin/
  value + termination/panic/recovery/interp all clean; 400k-iter fuzz clean).
- 2026-07-11 — **S-50 (b): comment inside interpolation is a distinct error.**
  doodle-rust `2aeb49c` (code) + `0a75678` (message aligned to the ratified
  wording); L§6.7 body rule `d96cc33`. Closes M1.4.
- 2026-07-11 — **M1.4: lexer strings/escapes/interpolation/bytes** (code).
  Spec gate `4501c00`. Code (doodle-rust commit below): `lex/string.rs`
  (structured string stream `StrStart (StrText | interp)* StrEnd`, recursion
  for nested interpolation with a depth cap; bytes as one `Bytes` token) +
  `lex/escape.rs` (closed escape set, shape-only; decode deferred to M1.6) +
  new diagnostic codes + `"`/`b"` dispatch. 8 `stage: lex` fixtures
  (L3.6.3/L3.6.5/L6.7) incl. an escape error *inside* an interpolation; suite
  13/0/3. 3-lens adversarial review (termination/recovery + escape/spec clean;
  2 minor interp findings): empty-interp false-positive on a token-less error
  body **fixed** (`saw_content` flag + regression test); comments-in-interp is
  **S-50** (provisional, user decision pending). Process note: a review agent
  reverted `tests/lex.rs` via `git checkout` mid-run — recovered; CLAUDE.md
  guard added (review agents run read-only).
- 2026-07-11 — **M1.3b: stage gate → Lex + conformance-runner execute/match
  path (atomic).** `stage::implemented_through()` → `Some(Stage::Lex)`; the
  runner now executes `stage: lex` tests instead of SKIPping, matching
  `expect-*` against real lexer diagnostics (new `matcher.rs`: errors are an
  order-insensitive set match on (substring, position) with no unlisted error
  and distinct per-expectation claims; warnings lenient). Structured
  `Expectation` model retained (was a bare count). Four `stage: lex` fixtures
  (valid forms; double-underscore, unterminated-string, emoji — each an
  expect-static-error; the emoji pins codepoint-column counting). Suite: 5
  passed, 0 failed, 3 skipped. Review done inline (the two review subagents
  were killed by a monthly-spend-limit API error mid-run): matcher set-match
  semantics stress-verified via a scratch suite; stale `stage.rs` M0 doc fixed;
  `num-001` given an `L3.6.2` secondary clause; empty-substring expectation
  now rejected. doodle-rust `7210c3b`.
- 2026-07-11 — **M1.3a: lexer core** (`crate::lex`) — tokens, numeric-literal
  shape (L§3.6.1/§3.6.2), plain-string boundaries, the S-2 newline/continuation
  state machine, UAX#31 identifiers, operator maximal munch, error recovery;
  diagnostics malformed-number + unexpected-character. Token-dump snapshots are
  the authoritative S-2 evidence (a mis-emitted NEWLINE is invisible at
  `stage: lex`). `lex()` requires load-normalized (CRLF→LF, NFC) source and
  debug_asserts it. No stage bump (that is M1.3b). 2-lens review (prior
  session): zero blocker/major; minors folded in (all-underscore message,
  bracket-depth note, NFC precondition, added coverage). Lexer spec gaps filed
  (inline-whitespace set, lone-CR). doodle-rust `898a602`.
- 2026-07-10 — **M1.2: source model — NFC, spans, positions (resolves S-1).**
  Spec: pinned S-1 in **L§3.1** (new "Source positions" paragraph), **L
  Appendix D.1**, and **E§8.1** (line/column span, code points). Code
  (doodle-rust `6aa0a7b`): `src/unicode.rs` (AD4 wrapper: NFC + UAX#31
  identifiers + module names) and `src/source.rs` (`Position`, `normalize`,
  `LineIndex` byte→code-point-column); `diag/render.rs` refactored onto it
  (snapshots byte-unchanged). Deps unicode-normalization + unicode-ident
  (license-clean). Conformance cases deferred to M1.3 (nothing lexable yet).
  2-lens review: one major (XID/ID divergence — filed + surfaced) + minors
  folded in. Two spec questions surfaced to the user (XID/ID; CRLF→LF).
- 2026-07-10 — **M1.1: diagnostics infrastructure + error-message rubric.**
  Code (doodle-rust `9c49651`): the `diag` module (`Diagnostic` /
  `DiagnosticCode` / `Note` / `Replacement` / `Suggestion` / `LoadError`) +
  a no-ANSI plain-text renderer (source snippets, carets, code-point columns,
  CRLF-safe, panic-free); 12 insta snapshots; CI green. Rubric
  (`plan/error-message-rubric.md`, discussions `96f0a4a`): drafted by the
  agent, **signed off by the user 2026-07-10**. Provisional choices D
  (structured `Replacement`) + F (code-point caret) **confirmed by the user**.
  2-lens review: two majors fixed (CRLF leak; test gaps) + minors. Four
  spec-deltas filed (see queue below).
- 2026-07-10 — **M0.9: M0 exit review — milestone M0 COMPLETE.** A 3-lens
  audit (exit-criteria / per-item acceptance / gap-hunt) independently
  re-verified all three M0 exit criteria and every M0.1–M0.8 acceptance
  against the tree at doodle-rust `95d3dc9`, all CI + hygiene green. Zero
  blocker/major. One spec-delta discovered in M0 (top-level `Completed`
  value, E§7.2, due M2a). Minor forward-notes recorded for M1 (conformance
  runner execute-path coupling; fuzz nightly pinning). Process points
  surfaced to the user: staticlib-drop → **resolved** (user chose static; capi
  is now staticlib-only, `036b615`); spec-delta escalation still open (issue/
  Appendix C now vs. batched curation).
- 2026-07-10 — M0.8: contributor docs + issue templates. `CONTRIBUTING.md`
  (build/test/hygiene, don't-game rule, spec-delta process, a two-tier
  review policy) in doodle-rust; GitHub issue forms — `bug` on doodle-rust,
  `spec-delta` on discussions (new `spec-delta` label created). Render lens
  confirmed both forms validate against GitHub's schema. 2-lens review: one
  major fixed — the review policy had contradicted the ratified M1.13/M1.1
  ("Reviewer = the user" for the message-quality bar). doodle-rust
  `95d3dc9`; discussions template in this commit.
- 2026-07-10 — M0.7: insta + fuzz plumbing. `insta` dev-dep + a committed
  deterministic snapshot of the M0.3 AST Debug (`tests/snapshots/`), run by
  ordinary `cargo test`; a `#[doc(hidden)]` `doodle_core::fuzz_smoke` seam
  and a detached `fuzz/` cargo-fuzz crate (own `[workspace]`; nightly-only,
  `cargo +nightly fuzz build` succeeds; not in CI at M0). A nightly
  toolchain (1.99.0) was installed locally for fuzzing; the engine stays on
  the stable pin. 2-lens review: no blocker/major; minor items M1-deferred.
  doodle-rust `8d789d9`.
- 2026-07-10 — M0.6: C ABI hello-world. `crates/doodle-capi`
  (`#[unsafe(no_mangle)] extern "C" doodle_version()` → NUL-terminated
  version via a process-lifetime `OnceLock<CString>`); cbindgen-generated
  committed `include/doodle.h`; `examples/c-host/main.c` smoke;
  `scripts/capi-header.sh` (currency check + `--write`, cbindgen pinned to
  0.29.4) and `scripts/capi-smoke.sh`; a `capi` CI job. All CI + hygiene
  green. 2-lens review: blocker (invalid `taiki-e/install-action@cbindgen`
  ref) + major (unpinned cbindgen → non-reproducible header gate) fixed.
  doodle-rust `bb56f4e`. **Revised `036b615`:** staticlib-only (the
  embedding form; C smoke links statically) — reverses the initial
  cdylib-only drop, per the user.
- 2026-07-10 — M0.5: wasm hello-world + size gate. `crates/doodle-wasm`
  (wasm-bindgen cdylib exporting `version()`); `scripts/wasm-size.sh`
  (build release wasm → `wasm-opt -Oz` → brotli → 300 KB budget, plan §6.5;
  fail-closed; `WASM_BUDGET_BYTES`-overridable); a `wasm-size` CI job that
  also self-tests the fail path. Hello-world ≈ 8 KB brotli. All CI +
  hygiene green. 2-lens review: one major (fail-closed) folded in.
  doodle-rust `ad8bbbe`.
- 2026-07-10 — M0.4: conformance test format + runner skeleton.
  `conformance/README.md` (ratified format v0, source of truth); a std-only
  `tools/conformance-runner` (discovers `.doodle` tests, parses/validates
  `#!` directives, SKIPs by doodle-core's `stage::implemented_through()`,
  prints `=== N passed, N failed, N skipped ===`); doodle-core `stage`
  module; four example tests (all SKIP at M0); a `conformance` CI job. All
  CI + hygiene green. 3-lens review: one major (a seed example misfiled
  under L6.2 vs L8.4) + minor robustness/fidelity items, all folded in.
  doodle-rust `2c8ae7e`.
- 2026-07-10 — M0.3: core pipeline skeleton in `doodle-core` — `span`,
  `diag`, `ast`, `machine` (`Value` per machine-design §3; `InstanceState`
  per E§3.3; result register + step cursor), `drive` (`Outcome` per E§7.2;
  `run()` drives a hand-built one-statement program to `Completed`). No
  parser, no machine-core mechanisms (M2a gate). Acceptance test observes
  `Int(42)` through the public API; hygiene + CI green. 3-lens adversarial
  review: no blocker/major; minor findings folded in. Spec-delta filed
  above (top-level `Completed` value). doodle-rust `c60c336`.
- 2026-07-10 — M0.2: build/test CI workflow — `.github/workflows/test.yml`
  with four jobs (`cargo test --workspace` on ubuntu/macos/windows +
  `cargo check --workspace --target wasm32-unknown-unknown` on ubuntu); the
  toolchain (channel + wasm32 target) is provisioned by an argument-free
  `rustup toolchain install` reading `rust-toolchain.toml`. All four jobs
  green on GitHub-hosted runners. doodle-rust `5354c38`.
- 2026-07-10 — Working plans `plan-m0.md`/`plan-m1.md` +
  `machine-design.md` drafted, adversarially reviewed (3 reviewers), and
  revised: machine-design v0.2 (unwinding redesigned around
  resolver-annotated exit targets; S-46 filed; S-9 resolution proposed),
  conformance format v0 gains stage/expect-warning directives, this todo
  file created.
- 2026-07-10 — M0.1: workspace + doodle-rust repos, hygiene checks + CI
  (green), MIT license (D-3), toolchain installed. Commits:
  workspace `a011bb7`, doodle-rust `a7ddf9c`, discussions `1b6d4ce`.
- 2026-07-09 — Implementation plan v0.1 (adversarially reviewed);
  D-1 resolved 07-10.
- 2026-07-04…09 — Language spec v0.1 (incl. string model, dict order),
  engine spec v0.1.
