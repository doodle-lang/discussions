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

(none — the protocol-member `end` ambiguity is **RESOLVED as S-52**, see
below; code follow-up outstanding)

**S-52 RESOLVED (user, 2026-07-11): every protocol member is terminated by
its own `end`.** Empty body (docstring permitted) = required; non-empty
body = default (no-op defaults via an explicit `nil`). Kills the silent
misparse structurally, makes "every `to`/`fn` ends with `end`"
exception-free, and turns bare signatures into a loud targeted error.
The §10.1-vs-App A `params?` divergence is reconciled (optional, per the
code). Spec landed (L§10.1 grammar + prose + Iterable example, App A,
App D.1, App C S-52). **Code follow-up: DONE (doodle-rust `bff4bcd`)** —
`proto_member` now requires the member `end` (empty body → required,
non-empty → default); `close_protocol` emits the targeted bare-signature
diagnostic; the ambiguity-pinning test is replaced with S-52 tests and the
L10.1 fixture updated. Read-only review: no defects.

**S-53 RESOLVED (user, 2026-07-11): single-line `"""x"""` is allowed** —
a triple-quoted literal closing on its opening line is a single-line form
(inline value, no margin rules, unescaped `"` OK); multi-line rules
unchanged; opener content without a same-line close is a static error
offering both fixes. The §8.6/§10.1 docstring examples are now correct as
written. Spec landed (L§3.6.4, App D.1, App C S-53). **Code follow-up
(next lexer session):** the single-line arm in M1.5's
`scan_triple_string` (+ `malformed-triple-quote` message update) + tests
(incl. `""""""` = empty, quote-runs at the closer).

## Awaiting the user (blocking)

(none — the M1.1 error-message rubric was signed off by the user 2026-07-10;
it stays open to revision and is exercised in earnest at the M1.13 review.

Context: plans ratified, machine-design v0.2 accepted, repos deliberately
public, issues enabled on discussions + doodle-rust: all 2026-07-10.)

**S-27 RESOLVED (user, 2026-07-11):** a lone-string body is the *result*
where the body produces a value (`fn`, named or anonymous) and the
*docstring* otherwise (`to`/module; record/protocol are docstring-only by
grammar). Rawness follows classification (docstrings raw; `fn` lone-string
result evaluated). Full rationale + rejected alternatives (incl. explicit
docstring syntax) in Appendix C S-27. **Code + L§8.6 edit: DONE** — parser
captures to/fn/module docstrings raw (fn lone string = result, via a
capture-then-rewind so a docstring's `{…}` is never parsed); L§8.6 states
the classification (discussions `2b0c4a0`); doodle-rust `387c9ed` (an
adversarial review workflow caught a CRITICAL — a first draft parsed the
docstring interpolation as code — fixed + regression-tested).

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

(none — M1.6 and M1.7 landed; see Done log. Next is M1.8.)

**S-51 RESOLVED (user, 2026-07-11): `(a) = 5` is accepted** — parens are
transparent around assignment targets (`'(' lvalue ')'` added to L§5.3 +
App A; rationale + language survey in App C S-51 and L App D.1). Matches
the shipped M1.7 parens-transparent parser; no code change. **Small
follow-up for the next parser work item:** a pinning accept-side test
(`(a) = 5` assigns to `a`, no diagnostic).

## Next up

Milestone **M1 — Front End** (see `plan/plan-m1.md`): M1.1 … M1.15, with
conformance tests landing at `stage: lex/parse` per item and upgraded to
`full` at the M1.10 checkpoint. M1 was blocked on M0.3/M0.4/M0.7 — all done.
M1.1–M1.7 (lexer, expression + statement parser, `stage: lex`/`parse`
conformance) have landed.

- [x] **M1.8 — declarations + docstrings — COMPLETE** (a/b/c1/c2 + the S-52 flip
      + the S-53 lexer arm; all in the Done log). (Boundary: `import`, call-site
      **block arguments** `f() do … end`, and the S-4 no-block-arg *enforcement*
      are **M1.9** — not M1.8; only the block *parameter* `do name` is M1.8.)

- [~] **M1.9 — imports + calls + blocks.** In progress; a and b landed
      (calls/block-arguments done); the milestone is essentially complete.
  - [x] **M1.9a** — `import` (L§11.2, five target forms + comma) with the S-7
        resolution-order note; a new `.*` (`DotStar`) token. Done log. doodle-rust
        `dcc0b63`. (DotStar ratified as **S-54**.)
  - [x] **M1.9b** — call-site **block arguments** `f() do (params) … end`
        (L§6.4/§8.5) on calls + the **S-4 no-block-arg enforcement** (`no_block_arg`
        flag set across a `while`/`with` header, cleared in every delimited
        sub-expression and nested body) + the enriched `stray_do` hint. Done log.
        doodle-rust `344cf32`. **Provisional block-params permissiveness spec-delta
        filed (needs a ruling) — see Awaiting-the-user.**
  - [x] **M1.8a** — `params`/`to`/`fn`/anonymous-`fn` (L§8.1/§8.2/§6.10). Done log.
  - [x] **M1.8b** — `record`/`ref record`/`protocol`/`extends`/`implement`
        (L§9/§10) + `capture_docstring`. Done log. (Its provisional protocol-`end`
        choice is superseded by the S-52 flip — MAJOR section, done.)
  - [x] **M1.8c1** — to/fn/module docstring capture per S-27 (raw, with the
        fn-lone-result rewind) + the L§8.6 edit. Done log. doodle-rust `387c9ed`.
  - [x] **M1.8c2** — `module`/`parameter`/`exports` + module-level-only placement
        rules (L§7.1, via a `nested` flag). Done log. doodle-rust `77ad3a5`.
        Read-only review: no defects.
  - [x] **S-53 lexer arm** — single-line `"""x"""` in `scan_triple_string`
        (speculative scan + rollback; a review-caught MAJOR fixed). Done log.
        The §8.6/§10.1 single-line-triple examples now lex as written.

- [~] **M1.10 — Resolver: scopes, slots, static-error battery.** In progress;
      **a landed**, b/c next. Design: `plan/resolver-m1.10-design.md`
      (concretizes machine-design §2/§6/§7/§12; S-5/S-6 rulings folded in). A read-only review caught 7 real design
      errors in the first draft (undeclared-assignment dropped from S-5; a
      `raise`-tail falls-off-end unsoundness; blocks-share-callable-slots vs. MD
      §8; construct-body scopes omitted; S-6 missing the field/index base operand;
      GlobalKind↔CellKind; a capture-chunk contradiction) — all fixed.
      **S-5 is RULED (user, 2026-07-16; full text in App C):** the doc's §4
      tail classifier + four-way lattice is ratified, with deltas — loop
      divergence is **lexical bound-`break` presence** (an M1.11
      exit-target lookup), *not* a reachability pass; condition-blind
      dead-tail rejection stated as a design property; `fn` bodies only;
      tail-`while` diagnostic suggests `loop`; S-6 shares the lattice.
      **S-6 is RULED (user, 2026-07-16; full text in App C):** the
      consuming-site model confirmed (producer-site blame in diagnostics;
      Void propagates through expression-position if/try to the outer
      consumer, never across an fn boundary); the static/runtime split as
      proposed; the site list gains `return`/`raise`/`break`/`continue`
      operands + parameter defaults, with dict keys/values and keyword
      args spelled out, under the invariant "every expression position
      consumes except the bare expression statement." **Companion rule 2a
      (new, load-bearing for the static subset):** declaration bindings
      (`to`/`fn`/`record`/`protocol`/`parameter`) are **non-assignable**
      — a static error in the const-reassignment family; add it to the
      M1.10b battery. REPL impact checked and banked into S-24
      (cross-increment redefinition = cell replacement; batch rules
      within a chunk; the runtime half of S-6 is the redefinition
      soundness backstop). The concrete-type gap-fills ([1]–[9]) had a
      second adversarial review (folded into the doc, `6aca132`): [1]/[3]/[4]/
      GlobalKind/BodyKind sound; [9] single-pass capture was **unsound**, fixed
      with a per-slot `cell_boxed` flag; added `slot_names` + decl-node slot
      resolutions + overflow-checked widths.
  - [x] **M1.10a — environment/name-resolution pass.** The `resolve` module
        (ResolvedModule + scope/frame model): local slots, `globals` (with
        declaration kind, for rule 2a), and reference classification
        (LocalSlot / BlockOuter static links / ModuleName free names) +
        `slot_names` + `stmt_spans`. Dispatch split into `walk/dispatch.rs`.
        Adversarial review: frame/hops classification, `home_fn`, `module_direct`,
        shadowing, determinism (Vec scopes, no HashMap) all sound; two MAJORs
        fixed (module `const` mis-tagged `Let`; param default saw sibling params
        — now enclosing-scope per L§8.2). 12 unit tests. doodle-rust `593b327`.
  - [x] **M1.10b** — static-error battery + exits. **DONE** (exits `2dccffc`,
        battery `d7e4cd4`, selective-import message `755f868`). Deferred: duplicate
        *params* (`to f(x,x)` / block `do (y,y)`) — no per-param node span; and
        wildcard-import provenance *naming* in the message (M5). 25 resolver tests.
    - [x] **exits** — exit-target annotation + placement (MD §12): a `Ctrl`
          stack (Callable/Loop/Block), `return`→HomeCallable (misplaced at module
          level), `break`/`continue`→ThisLoop/ThisBlock/ConsumerCall, callable is
          a barrier, `raise` not annotated; `exit_targets` side table +
          `misplaced-exit` diag. Review confirmed vs MD §12. doodle-rust `2dccffc`.
    - [x] **error battery** — duplicate-decl, const-reassign, rule-2a
          non-assignable declarations, and — **scope question RULED (user,
          2026-07-17; full text in App C S-39): neither option — the FULL
          assignment check is static now, wildcards or not.** Assignment
          targets differ from reads: since S-39 makes *every* imported name
          read-only, a name that isn't a lexically visible mutable `let` is
          an error whether it's undeclared *or* wildcard-supplied — the
          wildcard's opacity never changes the verdict, so no import
          resolution is needed and no no-wildcard special case exists.
          Rule: `name = v` errors unless `name` resolves lexically to a
          `let` (local/captured/module-level). Diagnostics: specific
          messages for const/declaration/selective-import targets now; the
          unknown bucket gets the dual-intent message (typo-`let` vs
          wildcard read-only). Only wildcard provenance-*naming* waits for
          M5. (AD5's linter delegation still governs unknown-name *reads*.)
  - [~] **M1.10c** — value discipline + captures + stage gate. In progress.
    - [x] **fn-falls-off-end (S-5)** — the four-way tail classifier
          (produces/diverges/value-less/indeterminate), `loop`-divergence via
          `loops_with_break`, `to`-call tail = Void, `fn` bodies only, tail-`while`
          suggests `loop`. Review confirmed sound; caught `return <void>` mislabel
          (fixed — `return expr` produces; its operand's Void-ness is S-6).
          doodle-rust `d82e134`.
    - [ ] **Void / S-6 consuming-site check** — the static subset (a `to` call
          used where a value is required — incl. `return`/`raise`/`break` operands,
          which S-5 defers here). Reuses the tail classifier's "produces" notion.
    - [ ] **`if`-expr-requires-`else`** semantic (part of the value discipline).
    - [ ] **closure captures** (resolve the deferred cross-`fn` refs, per B) +
          the `cell_boxed`/`CaptureSource` output.
    - [ ] **stage-gate bump Parse→Full** (+ conformance-runner Full executor) —
          the M1 staging checkpoint; do LAST (once the battery is complete so
          `stage: full` fixtures pass).
      **Capture representation RESOLVED: B** (user 2026-07-17, after adversarial
      review — resolver-design §8). Surfaced building M1.10a: a block nested in a
      closure referencing a captured var doesn't fit `Resolution::Capture(index)`.
      **B**: captures are cell-boxed frame *slots*; every ref is `LocalSlot`/
      `BlockOuter` + `cell_boxed`, `Resolution::Capture` dropped; a `CaptureSource`
      list drives closure creation. B is what machine-design §7/§8/§10 already pins
      (A — a separate capture array — would contradict §10). M1.10a deferred it
      (cross-`fn` refs sit in `deferred_captures`); M1.10c resolves them as B.
      **Flag for M2a (not resolver):** `Value` has no `Cell` variant but a
      cell-boxed slot must hold a cell ref — resolve at M2a (`Value::Cell` or a
      parallel `cells` vec).
      **S-11 is RULED (user, 2026-07-17; App C):** closures may mutate captures —
      by reference, sharing across captures of the same binding, const-ness
      travels, loop closures capture per-iteration bindings; M1.10c lands the
      L§6.10/§8.5 wording. (B satisfies S-11's sharing: one cell.)
      **Provisional (note, may want revisiting):** param defaults resolve in the
      *enclosing* scope (L§8.2 literal) — a default can't see a sibling param.

**Stage gate — now at Parse (M1.7):** `implemented_through()` returns
`Some(Stage::Parse)`; the conformance runner executes `stage: lex` and
`stage: parse` (matcher `run_static`, parametrized by stage — lexer vs.
`parse_to_diagnostics`). Future stage bumps (full/run) must likewise co-land
their executor arm in `tools/conformance-runner/src/matcher.rs` atomically.

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
App C; M1.5 lands the L§3.6.4 edit)**, **S-4 (M1.7, done — no-trailing-block
header parsing; L§6.4/§7.6 + App D.1)**, S-5 + S-6-in-full
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
  content line. **CONFIRMED (spec author, 2026-07-11): a decode-time error** —
  the forced intersection of S-49 (closed escape set) and S-3 rule 4 (no
  backslash-newline join), not a new fork. Code landed (doodle-rust `35229e3`):
  decode-time error, message upgraded to address both intents (literal `\` vs
  continuation), the last-content-line edge (`\` at end of value) covered, and
  single-line precedence pinned (unterminated-string wins; the triple case
  reports). **Bookkeeping owned by the spec author:** record as a clarification
  on Appendix C **S-49** (not a new S-number); the one-sentence L§3.6.3/§3.6.4
  pin rides with M1.5's spec batch.

Discovered at M1.6b (parser string decode; non-blocking):
- **NFC normalization of string-literal values** — L§3.6.3/§4.4 make a String's
  value NFC-normalized. The parser stores the decoded literal text in the AST
  *un-normalized* (`"e\u{301}"` → 2 code points, not composed `é`). This is fine
  at the AST layer — normalization is specified to happen at String
  *construction*, which the evaluator (M2) does (and re-does on concatenation) —
  but the literal→`String` construction must apply NFC (including to a pure
  single-text-part literal). Flagged so it isn't forgotten when eval lands.

Discovered at M1.7 (statement parser; non-blocking recovery-quality):
- **Noisy recovery on already-errored input** (carried from M1.6c/d + M1.7
  reviews): a missing statement separator, `a = b = c`, `with a = b, c = d do…`,
  and a missing collection separator each emit several diagnostics rather than
  one. All correct rejections and terminating — just noisier than the
  one-diagnostic discipline used for the depth bail. Tighten when convenient;
  the earnest message-quality pass is M1.13.
- **Cosmetic exit-span nit**: a bare `return`/`break`/… immediately followed by
  a non-boundary, non-operand token (e.g. `return)`) parses an operand, adding an
  "expected an expression" and stretching the exit node's span over the stray
  token. Malformed-input only.

Discovered at M1.9b (block arguments; awaiting a ruling):
- **Block-params permissiveness vs. the App A grammar (provisional).** The M1.9b
  parser accepts an empty `do () … end` (treated as zero parameters, same as
  omitting the `()`) and a trailing comma `do (a,) … end`. The Appendix A grammar
  `block-params = IDENT sep-by ','` (and the `( '(' IDENT ( ',' IDENT )* ')' )?`
  form) allows neither, but §6.4 prose says a zero-parameter block "may omit the
  `()`" (permissive-sounding) and the existing call-arg/function-param parsers
  already accept trailing commas — so the provisional choice keeps block-params
  *consistent with its siblings* rather than stricter. **Needs a ruling:** either
  align App A's grammar (permit empty `()` and/or a trailing comma, ideally
  uniformly with calls/params), or tighten the parser to reject them (accepting
  that block-params would then be stricter than function params). Pinned by the
  `block_arguments_l64_l85` test (`go() do () step() end` parses clean). No valid
  program's meaning changes either way; this is parse-surface only.

**S-54 RESOLVED (user, 2026-07-15): the M1.9a `DotStar` provisional is
ratified.** `.` immediately followed by `*` (no whitespace) is a single
`.*` token, not a continuation trigger — `import m.*` ends its line; bare
`*` (multiplication) still continues; `import m. *` is a parse error. No
valid program changes meaning. Spec landed (L§3.2 + §3.7 + App D.1 + App
C S-54); the `dot_star_is_one_token_not_a_continuation_trigger` test now
pins ratified behavior. **Small code follow-up (next lexer/parser
session):** the expression-position `.*` targeted diagnostic ("a field
access needs a name after the `.`").

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

- 2026-07-15 — **M1.9b: call-site block arguments (L§6.4/§8.5) + S-4 header
  enforcement; extract `parse/literal.rs`.** `Node::Call` gained `block:
  Option<BlockArg>` (params + body); `postfix.rs` `call()` parses a trailing
  `do (block-params)? body end` after the argument list. S-4 no-trailing-block
  mode: a `no_block_arg` flag set true only across a `while`/`with` header
  (`header_expr`), cleared in every delimited sub-expression (a `delimited()`
  wrapper: grouping, list, dict, index, call-args, string interp, **`if`
  condition**, **parameter default**) and every nested body (`enter_body`:
  `block()`/`body_with_doc`) — so `while f() do … end` is a while-body, and
  `while (f() do … end) do … end` runs the inner block on `f()`. `stray_do` now
  suggests parenthesizing. The string/bytes literal assembly moved out of the
  length-pressured `parse.rs` into `parse/literal.rs` (parallel to
  `collection.rs`). **Adversarial review (5-lens find→verify workflow, 18
  agents) caught two real false-rejections in the first draft** — the
  `no_block_arg` flag leaked into (a) an `if`-condition and (b) an anon-`fn`
  parameter default used within a header (both parsed via bare `expr(0)`); fixed
  by wrapping each in `delimited()`, with regression tests. Block-params left
  provisionally permissive (empty `do ()`, trailing comma) — spec-delta filed
  (see Awaiting-the-user). parse tests 38, full workspace green, conformance
  36/0/2 (+ L6.4 ×2, L8.5). doodle-rust `344cf32`.
- 2026-07-12 — **M1.9a: `import` declarations (L§11.2) + a `.*` (DotStar) token.**
  `Node::Import(Vec<ImportTarget>)` (path/wildcard/alias); `parse/moddecl.rs`
  `import_stmt`/`import_target` (five target forms + comma; `.*` not renamable;
  module-level-only). The parser records the dotted path only — module-vs-member
  is load-time (S-7, note landed in L§11.2). Also a new `DotStar` token fixing a
  real S-2 interaction (`import m.*` would merge with the next line because a
  bare `*` continues) — provisional, filed for a ruling (see Awaiting-the-user /
  the M1.9a discovered-delta). Read-only review: no defects. parse+lex 35,
  conformance 33/0/2 (+ L11.2 forms + wildcard-rename). doodle-rust `dcc0b63`.
- 2026-07-11 — **S-53: single-line triple-quoted strings + split `lex/triple.rs`.**
  `"""x"""` (closes on its opening line) is the single-line form (inline value,
  no margin, unescaped `"` ok, escapes+interp); multi-line unchanged; the hybrid
  (content after opener, no same-line close) is a `malformed-triple-quote` with
  both fixes. The triple-quoted machinery moved out of the over-length `string.rs`
  into `lex/triple.rs`. A read-only review (5.38M-input fuzz) caught a MAJOR in
  the first draft — a byte-level close pre-scan latched onto a `"""` inside a
  cleanly-closing interpolation and rewound the cursor (non-monotonic tokens);
  fixed by scanning the single-line form speculatively with the real
  `scan_interp` and rolling back on failure. Regression test asserts monotonic
  spans over the fuzz cases. Also updated the §8.6/§10.1 examples' fixtures to
  single-line. lex 25/0, conformance 31/0/2. doodle-rust `25c3588`.
- 2026-07-11 — **M1.8c2: module/parameter/exports + module-level placement**
  (L§7.1/§11.1/§5.5). `ModuleDecl`/`Parameter`/`Exports` nodes; `parse/moddecl.rs`.
  Placement via a `nested` Parser flag (set by `block`/`body_with_doc`, false at
  module level and in `module_body`); `require_module_level` reports a
  record/protocol/implement/module/parameter/exports parsed while nested
  (`let`/`const`/`to`/`fn` may nest; a module's contents are module-level).
  Read-only review: no critical/major/minor (nested save/restore sound,
  placement complete, termination/panic safe). parse 34/0, conformance 31/0/2
  (+ L11.1, L5.5, L7.1 placement). doodle-rust `77ad3a5`.
- 2026-07-11 — **S-52 flip: protocol members require their own `end`.** Replaced
  the M1.8b provisional (`proto_member` now parses a mandatory body + `end` via
  `body_with_doc(…, false)` — a member's lone string is its docstring; empty →
  required, non-empty → default); `close_protocol` emits the targeted
  bare-signature diagnostic; `ProtoMember` gains a `doc` span; S-52 tests replace
  the ambiguity pin; L10.1 fixture updated. Read-only review: no defects.
  doodle-rust `bff4bcd`.
- 2026-07-11 — **M1.8c1: S-27 docstring capture (to/fn/module).** A body's
  leading string is captured raw (`{…}` inert) and classified per S-27: a lone
  string is the result in a value-producing `fn` (capture-then-rewind so it is
  re-parsed as an evaluated literal), the docstring in a `to`/module. `Node::Module`
  now carries a `doc`. **An adversarial review workflow (4 lenses + verify)
  caught a CRITICAL** — the first draft ran a post-parse extractor, so a
  docstring whose `{…}` isn't a single valid expression emitted cascading errors
  and desynced (while the same text in a record was inert); fixed by capturing
  raw first (matching the record/protocol path) + a regression test over every
  body kind. §8.6 protocol-body omission (MINOR) also fixed. L§8.6 edit:
  discussions `2b0c4a0`. doodle-rust `387c9ed`.

- 2026-07-11 — **M1.8b: record/protocol/implement + docstring capture**
  (L§9.1/§10.1/§10.2). `Record`/`Protocol`/`Implement` nodes + `ProtoMember`;
  `parse/typedecl.rs`; `capture_docstring` (consumes a string's token stream
  without parsing its interpolations → raw span; `Callable.doc` is now
  `Option<Span>`). Records: fields + docstring-only body (extra content → error).
  Protocols: `extends`, required vs default members. Implement: reuses
  `callable_decl`. **Read-only review found a MAJOR: the L§10.1 protocol-member
  `end` ambiguity** — provisional choice shipped + pinned + surfaced (see MAJOR
  section). `capture_docstring`, `field_list`, record parsing all verified sound
  (400k-style probes; termination/panic/depth clean). Discovered the single-line
  triple-quoted docstring spec inconsistency (see Awaiting-the-user). parse 28/0,
  conformance 27/0/2 (+ L9.1×2, L10.1, L10.2). doodle-rust `a78b86d`.
- 2026-07-11 — **M1.8a: to/fn declarations, anonymous fn, shared params**
  (L§8.1/§8.2/§6.10). `Node::Callable { kind, name, params, body, doc }`, `Param`
  (Ordinary+default / Block `do name`), `CallableKind`. `parse/decl.rs`
  (`callable_decl`/`anon_fn`/one `param_list`; block param last, at most one;
  defaults). `fn name(…)` = decl, `fn(…)` = anon (one-token lookahead). Read-only
  review (fn/anon disambiguation, block-param-last, default-expr boundaries,
  param_list termination, depth): no critical/major; folded one cosmetic MINOR
  (dead index → bool). parse 23/0, conformance 23/0/2 (+ L8.1, L8.2). doodle-rust
  `90d002c`.
- 2026-07-11 — **M1.7: statement parser + stage → Parse** (L§7). Recursive-
  descent statements on the Pratt expression parser: `ast` statement nodes
  (Let/Const/Assign, Block, If, While, Loop, With, Try, Return/Break/Continue/
  Raise); `parse_program`/`parse_to_diagnostics`; `if`/`try` as expression
  primaries (same node in expr/stmt position); `parse/stmt.rs` (bodies,
  separators, let/const, assignment + lvalue check, exits) and
  `parse/control.rs` (if with else-if flattening + nearest-`end` else binding,
  try/rescue, while/loop/with). Shared `guard_depth` gives bodies + expressions
  one MAX_DEPTH budget (no bypass). **S-4** resolved: header expressions parse in
  no-trailing-block mode + a `stray_do` diagnostic for a `do` that starts a
  statement; spec pinned (discussions below). Stage gate → `Parse`; matcher
  `run_lex` → stage-parametrized `run_static` + `parse_to_diagnostics`; four
  `stage: parse` fixtures (L3.2 bumped, L5.3, L7.1, L7.6). Two read-only reviews
  (parser correctness — 400k-input fuzz + 1–8 MB deep-nest stack tests, no
  panic/overflow; integration + spec fidelity): no critical/major; folded the
  one actionable MINOR (softened the `stray_do` message so it doesn't advertise
  the M1.8-only parenthesized escape hatch). Filed the parenthesized-lvalue
  spec question (above) and the recovery-noise/exit-span nits (below). Suite:
  parse 20/0, conformance 21/0/2, hygiene 6/6. doodle-rust `962d207`.
- 2026-07-11 — **M1.6d: list and dict literals** (L§4.7/§4.8). `List`/`Dict`/
  `DictEntry`/`DictKey` nodes + `parse/collection.rs` (`list_lit`/`dict_lit`;
  bare-word key = string key via the shared `at_ident_colon`; computed keys;
  trailing commas; `{}` = empty dict). Read-only review: SHIP; folded in its
  finding — a stray closer `)`/`]`/`}` is no longer swallowed by `primary`, so
  enclosing collections recover to the right shape. doodle-rust `58f6d51`.
- 2026-07-11 — **M1.6c: postfix operators — access/index/calls + keyword args**
  (L§6.3/§6.4). `Field`/`Index`/`Call`/`Arg` nodes + `parse/postfix.rs`
  (`postfix_chain`, tightest-binding, left-assoc; keyword args via `Ident :`;
  positional-before-keyword enforced; trailing commas). Read-only review: SHIP
  (precedence vs L§6.5 all correct, no panic/loop — flat chains loop, nested
  route through MAX_DEPTH); the user-visible `a.1` double-report fixed. Split
  postfix into a submodule to keep `parse.rs` under the length limit. doodle-rust
  `2e0b386`.
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
