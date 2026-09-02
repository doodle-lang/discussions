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

**Three bugs found by the M4.10 multi-lens exit review — all FIXED (2026-08-26,
doodle-rust `61ca2a2`).**
1. **Unwind: the handling stack leaked on non-raise exits from a rescue body.**
   `unwind/cleanup.rs::discard_cont` popped `WithRestore` but not `PopHandler`, so a
   `break`/`continue`/`return` out of a rescue body left its handling entry — a later
   bare `raise` re-raised the **stale** exception (and `machine.handling` grew
   unbounded). Fix: `discard_cont` now pops `PopHandler` too, mirroring `raise_unwind`.
   Regression: `L12.2/rescue-007`.
2. **Dict: a value-record key was not copied on store.** `dict::insert` copied the
   value but stored the key verbatim, so a value-record key **aliased its source**;
   mutating the source (`k.n = 9`) desynced the stored key from its hash bucket →
   `KeyNotFound` + a **determinism leak** (lookup depends on post-insert mutation). Fix:
   `copy_on_bind` the key on the new-entry branch. Regression: `L4.8/key-005`.
3. **Float literal overflowing f64 became `Float(∞)`** — the S-56 finite-float invariant
   violated at the source boundary (`1e400`). This was the deferred L§3.6.2 front-end
   item (M2a.3a's `#[ignore]`d tripwire). Fix: `lower_float` rejects an overflow-to-∞
   literal with a new **front-end** diagnostic `float-literal-out-of-range` (not an S-58
   runtime slug); underflow (`1e-999` → 0.0/subnormal) stays legal. Tripwire un-ignored;
   regressions: `L3.6.2/float-001` (static error), `float-002` (underflow legal).
   Non-bug flagged for M9a: `-0.0` renders as `"-0.0"` while `-0.0 == 0.0` (a `Stringable`
   question). The hashing/seam lens found no bugs (hash/`==` coherence + the seam fix
   verified solid).

**`seam_concat` missed non-Hangul backward-composing starters → non-NFC results —
FIXED (M4.10, 2026-08-26; found same day by the M4.10 UCD vectors).**
`unicode::seam_concat` (the AD4 seam optimization behind string `+`, `*`, and
interpolation) extended its seam region across a leading **Hangul** V/T jamo in `b`
only (`is_backward_composing_jamo`); any *other* backward-composing starter — e.g.
Sinhala `U+0DD9 + U+0DCF → U+0DDC` — was treated as a clean boundary, so the seam
composition was skipped and the result **not NFC**. Impact would have been a debug
panic (`alloc_string`'s `is_nfc` assert) or, in the release wasm demo, a silent non-NFC
string breaking the NFC invariant (`==`, dict keys, grapheme length, replay) for
Brahmic scripts with composing vowel signs. **Fix:** replaced the hardcoded jamo range
with the general **NFC_QuickCheck = Maybe** predicate (`composes_backward`), which is
exactly the canonical-composition second-element set — it subsumes Hangul V/T (Maybe)
and excludes Hangul L / plain starters (Yes), and chaining is automatic (any char that
composes onto the composed prefix is itself Maybe). **Verified:** the UCD
NormalizationTest seam pass (`tests/unicode_ucd.rs`, ~28k splits over 19k rows) is green,
plus targeted Sinhala cases in `unicode.rs`'s `seam_concat_equals_whole_string_nfc`.

**M4 survey (2026-08-23): `[1] == [2]` panicked the engine — FIXED (M4.0,
doodle-rust `9ee3d9e`).** Lists became constructible at M2b.5a, but the compound
arm of `equal` was `unimplemented!`, so any `==`/`!=` on two lists aborted
(release wasm is `panic = "abort"`) — legal Doodle code crashing the tab.
**Fix:** M4.0 implemented the list arm of structural `==` (L§4.13) with a
cycle-safe `Vec` pair-stack (no hasher). Tests: 6 `mode: run` `L4.13` fixtures
(incl. `[1] == [2]` → false) green on native + through wasm, plus a heap-built
cycle-safety unit test. Dict/record arms stay for M4.4.

**M3.6 review (2026-08-21): resolve-with-raise on a *cancelled* suspended
instance yielded `Raised`, not `Faulted(Cancelled)` — FIXED (user chose "fix
the engine now", doodle-rust `6ab0927`).** `resolve_slice`'s `Resolution::Raise`
arm short-circuited to `Outcome::Raised` without driving, so it never polled the
cancel token; the `Resolution::Value` arm drove and faulted `Cancelled`. Hence
`cancel()` + resolve-with-value → `Cancelled`, but `cancel()` + resolve-with-raise
→ `Raised`. (E§10.1 permitted `Raised` as a post-cancel outcome, so it was not a
violation — but the asymmetry was a wart.) **Fix:** when a cancellation is pending
at a `Raise` resolution, discard the pending request and arm the cancel unwind
(`Instance::discard_pending_and_cancel`), then drive — the stack tears down to
`Faulted(Cancelled)` WITHOUT running the parked call's continuation, so (unlike
unsticking with a fabricated value) the discarded resolution has no program-visible
effect. The value arm is unchanged. **Tests:** doodle-core `drive_directives`
(raise+cancel → Cancelled, no output; value+cancel-with-work-remaining → Cancelled
after the resumed statement); doodle-web `pump.test.mjs` e2e (handler throws + abort
→ `cancelled`). **Spec:** E§10.1 S-23 pin updated (cancel wins over a host raise).
Reachability was narrow (a handler that *throws* while the stop signal is aborted);
M3.6's turtle handler returns normally on abort, so it was already correct.

(The protocol-member `end` ambiguity is **RESOLVED as S-52**, see
below; code follow-up outstanding.)

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

(none)

**M1.13 COMPLETE (doodle-rust `9af5f7b`).** User sign-off 2026-07-19 (all 41 rows
approved as tabled; a NEEDS-WORK/FAIL sign-off endorses the verdict + intended
fix), then the in-scope work landed: (1) the **span fixes** — unclosed-construct/
-delimiter errors now point at the opening token (systematic across all 15 sites;
a review caught two hand-rolled ones — group `(`, interpolation `{` — folded in) +
the `26` "original is here" note; (2) **corpus additions 42–45** (interpolation/
escape diagnostics). Final table **30 PASS / 7 NEEDS-WORK / 8 FAIL** (45 programs),
every diagnostic on the correct span. Still spun off (not gating): the `25`
wildcard-only hedge (S-39 one level deeper), `05`'s stray-`do` misdiagnosis (→
M1.9b), cascade suppression, de-jargon, rubric Appendix-A reconciliation.

**S-55 RESOLVED (user, 2026-07-18) — the twice-flagged §8.7 item, closed
with an extension:** procedure tail positions confirmed AND (surfaced in
review of the question) **frame reuse requires kind match** — naive
mixed-kind reuse is semantics-visible (to-tail-fn leaks the value past
Void completion; fn-tail-to bypasses the falls-off error), so mismatched
marked-tail calls run as ordinary calls; sound loops are kind-pure
(every unbounded mixed cycle contains an fn→to falls-off edge). Landed:
L§8.7 + App D.1 + machine-design §11 v0.2.1 + App C S-55. **M1.11
marking is unaffected** (positional; the kind check is apply-time).
**M2a follow-up:** the reuse kind-check + mixed-kind parity conformance
tests.

(The M1.12 golden-corpus decisions are RESOLVED (user): #44 wrapped (`…` →
`show(i)`); corpus lives in doodle-rust with a sync-check script, CI wiring
deferred. Both shipped in doodle-rust `175de57`.)

(The M1.1 error-message rubric was signed off by the user 2026-07-10;
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

**Milestone M4 — Language completion I (data, errors, strings) — IN PROGRESS
(started 2026-08-23).** Working plan **`plan/plan-m4.md`** written + **4-lens
adversarial-reviewed + revised**, **awaiting user ratification** before
implementation. A read-only survey (confirmed by the review) established the
ground truth: the front end is **complete** for every M4 construct; **all M4 work
is machine-layer**, behind several `unimplemented!` sites (`step.rs:300`/`:429`/
`:358`, `control.rs:114`, `compare.rs:101`). Decomposed into **M4.0–M4.10 across
three tracks** (data / control / strings); the review reshaped it: **M4.0** (list
`==`, closes the panic first), **M4.5a/b/c** split (unwind foundation / try+
exceptions / trace — decouples `with` from the exceptions decision), **M4.8a/b**
split (graphemes / bytes), plus string `*` and record `is` re-added.
Five decisions for the user (plan §Decisions): **D-M4-1** record-as-key model
(S-29 — **RESOLVED + spec landed 2026-08-24**), **D-M4-2** engine error-value shape (E§9 — **RESOLVED + spec landed
2026-08-24**: one built-in `Error(kind, message, details)` record, App C S-58), D-M4-3 Unicode pin (17.0-if-supported-else-
16.0) + `unicode-segmentation`, **★ D-M4-4** R8 magnitude cap, D-M4-5 S-37
type-value spellings (**`Procedure` half RESOLVED 2026-08-25: the
`Callable`/`Procedure`/`Function` split**, App C S-37). **Key review corrections:** the `WithRestore` continuation
is **unbuilt** (not "inert since M2a.6"); S-24→**S-28** (the dict-hash delta);
cross-kind hash/`==` consistency (`{1:"a"}[1.0]`) is a named M4.1 requirement.
**Progress:** **M4.0 DONE** (`9ee3d9e`) — list `==`, closed the `[1]==[2]` panic.
**M4.1 DONE** (`b9b73e8`) — the engine's first hasher (hand-rolled fixed-key
SipHash-1-3, `==`-coherent so `{1:"a"}[1.0]` hits) + heap dicts (insertion-ordered,
first-key-wins, GC-traced), `{k:v}` literals + `d[k]` reads; 6 L4.8 fixtures green
native + wasm. **D-M4-1 RESOLVED (user):** scalar keys only for M4.1; record keys
(reachable-immutable) defer to M4.4. **M4.2 DONE** (`5227f91`) — records: type
value (`TypeObj` now builtin-or-record), `record … with … end` decl, constructor
(`Point(x:3,y:4)`), field read (`p.x`), nominal `is`, GC; 6 L9.1 fixtures green
native + wasm. **Copy-on-bind moved to M4.3** (unobservable without mutation).
**M4.3 DONE** (`046b1a6`) — place chains (`a.b.c = v` / `d[k] = v`, Field/Index
targets) + value-vs-`ref` copy-on-bind (L§4.14/§5.3/S-38); +12 fixtures green
native + wasm. List *index* places (`xs[0]=v`) ride in with M4.8 list indexing;
copy-vs-navigate covered here via records + dicts. Also split `step.rs`→`eval.rs`
and `heap.rs` tests→`heap/tests.rs` (both under the 500-line soft limit).
**M4.4 DONE** (`8ac8feb`) — structural `==` dict/record arms (cycle-safe memo
generalized to lists/dicts/records; dict `==` order-independent; record `==` nominal)
+ record dict keys per the S-29 ruling (value record with recursively-hashable fields
hashes structurally; non-hashable key raises naming the offending field; hashing never
enters a reference graph). +9 fixtures green native + wasm. Split `compare.rs` tests to
`compare/tests.rs`. **M4.5a DONE** (`7185403`) — unwind-cleanup foundation: the
`WithRestore`/`TryHandler` cleanup-cont category + `dyn_stack`, every exit runs
`WithRestore` as it pops, and raises unwind through the frame channel (`Unwind::Raise`,
running cleanup) instead of straight to the boundary. `TryHandler` catch is the M4.5b
seam. Split unwind.rs → `unwind/{cleanup,arms}.rs`. **M4.5b DONE** (`238cdf5`) —
try/rescue/raise + exceptions-as-values (D-M4-2/S-58): engine raises materialize a
built-in `Error(kind, message, details)` record (details `{}` for now), `rescue e` binds
it, `e is Error`/`e.kind` inspect it, `raise value` throws any value, bare `raise`
re-raises with the original trace, `try` is an expression. New modules `exception.rs` +
`protect.rs`. +9 fixtures green native (134) + wasm (62). **M4.5c DONE** (`f96e4b2`) —
trace capture: `Trace` gains live `frames` (call-site span + tail_count) + `tail_elided`
history, captured at the raise site (`observe::capture_trace`), deterministic, no handles.
Completes the M4.5 error trio. **M4.6 DONE** (`0cda7a9`) — `with`/`parameter` runtime
(new `machine/dynamic.rs`), the M4.5a `WithRestore` producer end to end: restore on every
exit tier + cancel-mid-`with` (accept #5); `Frame.dyn_depth` per-frame exposure; S-10
runtime half (`no-value-destination` at the break site + S-46 native parity). Two ratified
extras: the S-5 `with`-tail fix (a diverging `with` body no longer reads as fall-off-end)
and the `with`-target static check (`with-target-not-parameter`, §5.5). Native 143, wasm 69.
**M4.7 DONE** (`3bf72ec`) — string `+`/`*` (new `machine/strop.rs`) off `arith`'s
number-only path; the AD4 seam pass is the reusable `unicode::seam_concat` (renormalizes
only the boundary; verified equal to whole-string NFC exhaustively). `*` symmetric (`Int`
either side, S-59): `Float`→type-mismatch, `0`/empty→`""`, negative→`negative-count` (new
slug), over-limit→`LimitExceeded` (via generalized `pending_fault`; R8 interior cap stays
M4.10). Native 153, wasm 79. **Deferred to its own chunk (user, 2026-08-25):** the
string-churn criterion benchmark (no suite/dep exists yet). **S-30** rides M4.8b.
**M4.8a DONE** (`b5f99b5`) — graphemes + runtime indexing. `unicode-segmentation` 1.13.3
(**D-M4-3 confirmed UCD 17.0**, all three crates aligned, joins the compile-time cross-
check); lazy grapheme memo on `StrObj` (pure cache excluded from `bytes_allocated`, MD §5);
`s[i]`/`list[i]`/`bytes[i]` indexing (`Node::Index` arm), out-of-range/negative → new
`index-out-of-range` slug (sign-branching message); `length` + `each`-over-graphemes
(provisional intrinsics, native + wasm demo registries). Native 161, wasm 87. **Carry-
forwards:** `index-out-of-range` `details: {index, length}` rides the message-rubric /
structured-details work (`make_error` is `{}` for all kinds today); list element assignment
`xs[i] = v` is a separate pending list-mutation item. **M4.8b DONE** (`e93adb8`) — bytes↔string
bridging: Doodle `encode`/`decode` (provisional intrinsics; `encode` can't fail, `decode`
validates UTF-8 + NFC, malformed → new **`invalid-utf8`** slug with byte position in the
message; round-trip law). **S-30 resolved** — host `make_string` keeps its error-return
`ValueError::InvalidUtf8` but carries the same position (one story). Native 166, wasm 92.
Carry-forward: `invalid-utf8` `details: {position, byte}` rides the message-rubric work.
**M4.9 DONE** (`e805a4f`) — string interpolation runtime + the D-M4-5/S-37 type-value
split. Interpolation (`"…{expr}…"`) evaluates: each `{expr}` is driven through
`Cont::StrInterp`, rendered by the placeholder `Stringable` dispatcher, and folded with
the literal runs via the AD4 seam routine (a `seam_concat` left-fold == a single NFC
pass). The dispatcher is invoked **directly** (a hidden binding, not the name
`to_string`), so a local `to_string` can't change interpolation (L§15 hook 1). The
provisional `render()` retired into a new `machine/stringify.rs` shared by `print` +
interpolation. **S-37 Procedure half** (spec `268c822`): the callable trio replaces the
single `Procedure` type value — `Callable` umbrella (any callable), `Procedure` (`to`),
`Function` (`fn`); foreign callables classify by descriptor (`print` is a Procedure);
type values are not `Callable`; M2a.5 `is` tests flipped. Native 173, wasm 98. **Carry-
forwards:** compound-value rendering (`<list>`/`<dict>`/`<record>`/…) stays a native
**placeholder** — the real `to_string` output is M5 (protocol dispatch) / M9a (stdlib
defaults), carved out of M4 acceptance; the runtime `take_value` backstop in `str_interp`
covers only the Void-in-interp cases the resolver can't catch statically (the S-6 static
check already rejects a lexically-known `to` call in `{…}`).
**M4.10 IN PROGRESS** (convergence). **R8 magnitude cap DONE** (`a197e13`, D-M4-4):
interior mid-op polling proved infeasible (atomic `num-bigint` calls; `**` is ≤32
squarings dominated by the last few huge multiplies), so `*`/`**` are guarded
**before** running — a pre-size estimate faults `LimitExceeded(Heap)` before
allocating (uniform with string `*`), and a pre-charge bills result bytes against the
step budget (`FusedCounter::charge`) so a huge magnitude faults `StepBudget` under a
bounded budget; both deterministic upper bounds. Default heap 16 GiB → 1 GiB; the wasm
demo loads with 64 MiB. `Machine::for_test()` (in the length-exempt `machine/tests.rs`)
lets arith unit tests supply a throwaway machine. **Spec RATIFIED (user, 2026-08-25):**
E§10.2 step budget now counts **work units** (result-growing ops pre-charge a size
estimate); E§10.1 rails diverge (fuel = statement safe points, budget also carries op
charges); when an op trips both rails the **heap** fault is reported. App C S-20/S-40
bracketed. The "cancel mid-operation" conformance idea is dropped as unreachable for
atomic ops; the deterministic pre-op fault is tested instead. **Also (S-12 closes /
`exponent-too-large` RETIRED):** a `**` too big to store — including a u32-overflowing
exponent — is now a magnitude *fault*, not a raise; `exponent-too-large` removed from
the `ExceptionKind` enum + S-58 catalog; `|base| <= 1` computes trivially whatever the
exponent (`1 ** huge == 1`, which previously wrongly raised). **UCD conformance vectors
DONE** (`8daeaeb`): official Unicode 17.0 `NormalizationTest.txt` + `GraphemeBreakTest.txt`
vendored under `crates/doodle-core/tests/data/ucd/` (~2.9 MB, Unicode license header
kept); `tests/unicode_ucd.rs` checks `nfc`/`grapheme_offsets`/seam-law/version against
them — the seam pass (~28k splits) **caught the MAJOR `seam_concat` NFC bug** above,
now fixed (NFC_QuickCheck=Maybe backward-composer predicate). **Full-suite determinism
gate DONE** (`8463ebc`): two new GC-stress double-run gates — one over the M4 feature set
(records value/ref + place chains; dicts / fixed-key-SipHash hashing over string/50-int/
record/cross-kind keys; strings seam/repeat/interp/index; exceptions; with/parameter),
one over the R8 magnitude faults (heap-estimate + step-budget-charge fault at the same
step under GC pressure). Criterion #6 (hashing must not leak nondeterminism) covered.
**Exit review DONE** (`61ca2a2`): the exit-criteria walk (#1–#6, all covered; +5 fixtures
crisply stating #2/#3/#4 — `nfc-001`, `seam-005`, `rescue-006`), a 3-lens adversarial
review (unwind/exceptions, place-chain, hashing/seam) that **found + fixed 3 MAJOR bugs**
(see the MAJOR section: handling-stack leak, dict-key-not-copied, float-literal-∞), and
App C reconciliation (S-38 closed; S-37 Procedure-half + Stringable-dispatcher pin closed,
spellings stay provisional; S-56 front-end literal-diagnostic line closed; S-10/S-28/S-29/
S-30/S-58 already resolved). Native conformance 180, wasm gate 104.

**★ MILESTONE M4 — Language completion I (data, errors, strings) — COMPLETE
(2026-08-26; M4.0–M4.10 all landed + exit-reviewed).** Records (value/ref, place chains),
dicts (fixed-key-SipHash, record/cross-kind keys), structural `==`, the unwind foundation
(try/rescue/with/parameter/cancel), exceptions-as-values (`Error` record), strings
(grapheme-indexed, NFC, AD4 seam `+`/`*`/interpolation, bytes bridging), the R8 magnitude
cap, the Callable/Procedure/Function split, and UCD 17.0 + determinism gates all ship.
**★ MILESTONE M5 — Modules, imports, protocols, prelude `[L]` — STARTED (2026-08-26).**
Working plan written: **`plan/plan-m5.md`** (chunks M5.0 multi-module refactor → M5.1 load
state machine → M5.2 imports/aliasing → M5.5 protocols/dispatch → … → M5.10 exit review;
critical path + parallel tracks noted). **The one fact that dominates M5:** the front end
already parses the full grammar (`Protocol`/`Implement`/`Exports`/`Import`/`Module` AST
nodes exist), so M5 is a **resolve + machine + drive-layer** milestone whose real cost is
the single-module → multi-module refactor (M5.0). **★ decisions pending (D-M5-1…D-M5-6):**
Stringable/Hashable M5-vs-M9a split, resolver sync/async (E§6, blocks M5.1), S-44
native-module parameter cells, `extends` depth, `module`-block form, S-43 shadowing warning.
One M4 carry-forward: the S-37 type-value *spellings* stay provisional (user's call).
**M5.0 CORE DONE** (`6815e45`): the machine is module-aware — `Instance.modules` table,
per-frame `Frame.module: ModuleId`, `step` derives the executing frame's module's
resolved+namespace; main module reframed as `ModuleId(0)`, all gates green (conformance
180, wasm 104). Deferred to their exercising chunk: cross-module call lookup + multi-
namespace GC rooting → M5.1; `CellObj` kind/provenance → M5.2/M5.5. **Next: M5.1** (resolver
hook + load state machine + suspendable driving — **D-M5-2 RESOLVED 2026-08-27:
import is a suspension, async-capable from v0.1**; App C S-60).
**M5.1a DONE (2026-08-27): the import-load machinery.** `import` **suspends** with
`SuspendedImport(ImportRequest{path, importer})`, resolved via `resolve_import(Source |
NotFound | Raise)` (reuses the M2b.4 park/resume); the `{loading, loaded, failed}` state
machine + `by_path`/`by_canonical` singleton caches; a sub-module's top level drives as
**ordinary frames** (it can itself suspend); **circular-import** naming the cycle; **S-8
failed** — `LoadState::Failed(value)` retains the load's exception (a GC root), re-import
re-raises it unchanged (latent single-run: a load failure is uncatchable — `import`-in-
`try` is a static error — so it terminates); **module-load-error** for a fetched module
with static errors; the **multi-namespace GC rooting** the 2nd module makes live (all
modules' namespace cells are permanent roots). Two S-58 slugs ratified: `circular-import`,
`module-load-error`. Tests: `tests/modules.rs` (10) + machine unit (retention, GC-rooting).
All gates green (conformance 180, wasm 104). **Deferred: cross-module call lookup + the
heap module value + name binding of every import form → M5.2** (the module value gets a
consumer once binding exists). **Tracked obligation (rubric):** per-kind `details` schemas
must be populated for every `Error` kind before M6 (the IDE consumes `details`); messages
are never API.
**M5.2a DONE (2026-08-27): the multi-module machine + bare-module import.** The
module-table threading (`step`/`dispatch`/call path/reentrant nested drive take `&mut
[LoadedModule]`; executing module = `resolved.canonical_id`); the **cross-module call
fix** (`apply` reads the callee's params/defaults/slots/body from *its* module's AST, the
caller's from the call site — this was the M5.0-deferred "cross-module call lookup");
**`import m` / `import m as y`** binding (module value → a new namespace cell, added to the
permanent GC roots); **`m.x` member access** (L§4.11 — a member is one of `m`'s own
module-level defs; prelude names are not members; reads the live cell). `Value::Module`
was already present. Tests: cross-module call + defaults + live member read; `import`/`as`;
missing-member + prelude-non-member raises; bound-cell-survives-GC. All gates green
(conformance 180, wasm 104).
**M5.2b DONE (2026-08-27): S-7 dotted-path + member imports + cell aliasing.** S-7: the
loader tries the whole path as a module first; a path the host resolves `NotFound` (a
`not_modules` cache) falls back to member (`import a.b` → member `b` of module `a`); a
single-segment miss still raises `module-not-found`. Member imports (`import m.x` / `as`)
bind by **aliasing the exporter's cell** (AD5) — the importer's name maps to `m`'s existing
binding cell, so reads are live (read-only; assignment already a static error, S-39 core);
a non-member raises `no-such-field` at the import. Split `drive.rs` → `drive/config.rs`
(length). All gates green (conformance 180, wasm 104).
**M5.2c DONE (2026-08-27): wildcard imports + S-13 ambiguity — M5.2 COMPLETE.** `import
m.*` records a wildcard source (deduped) on the importer; a free name not in the namespace
**resolves on use** across the wildcard sources' exports (AD5) — 0 = undefined, 1 = live
alias of the exporter's cell, **2+ = `ambiguous-import`** raised at the use site naming both
modules (import order). Explicit/selective imports + local decls sit in the namespace, so
they win for free (S-13 override). Threaded `modules` into the read path (`step` →
`eval::eval` → `read_ref`). Slug `ambiguous-import` **ratified** (App C S-58 + rubric App A:
`details {name, modules:[a,b]}`, raised at use). Tests: wildcard-all-exports, live alias,
explicit-override, local-shadow, two-wildcard ambiguity (**exit #3**), undefined-name. All
gates green (conformance 180, wasm 104). **S-13 closed.** **Residue:** S-39
wildcard-provenance-naming for *assignment* (polish, non-gating).

**M5.5a DONE (2026-08-27): protocol dispatch core.** The `CellObj` **`kind` tag** (AD5,
deferred since M2a) is now built (`CellKind::{Let, Const, Parameter, Dispatcher(member)}`,
mapped from `GlobalKind` at seed/bind). Protocol **registry** on the machine (interned member
names, protocol defs, `implement` blocks — index-addressed `Vec`s, no hashing, load-order
numbering for replay); `protocol`/`implement` register at load (`machine/protocol/`).
`protocol P` binds `P` to a `TypeKind::Protocol` value and binds each member name to a
**dispatcher cell** in the namespace. A bare member call **binds args against the member
signature (S-31 — positional or by the member's keyword), dispatches on the first
parameter's runtime type** as an ordinary driven call, and enters the impl (or a default),
raising **`protocol-not-implemented`** (type implements no supplying protocol) or
**`ambiguous-member`** (two unrelated implemented protocols supply the name). Qualified
`P.member(args)` via field access on a protocol value. Umbrella `implement … for Number`
expands to leaves. GC roots the registry's default/impl callables. **`x is P` landed here
too** (M5.6 absorbed — trivial with the registry). Both slugs **ratified** (user 2026-08-27;
App C S-58 + rubric App A: `{type, protocol, member}` / `{member, protocols:[a,b], type}`).
Files split to stay under length: `machine/protocol/{mod.rs,load.rs}`, `machine/cancel.rs`
(extracted `CancelToken`). Tests: `tests/protocols.rs` (10) — dispatch-by-type, default,
override, not-implemented (**names type+protocol+member+fix**), ambiguity + qualified
disambiguation, keyword dispatch (S-31), `is P`, block-arg dispatch, incomplete-impl. All
gates green (native 179 lib + 10 protocol, conformance 180, wasm32 check, hygiene 6/6).
**Provisional → M5.5b:** first-param-no-default not yet *enforced* and member-param defaults
not yet evaluated (every ordinary param must be supplied); a bare dispatcher value classifies
as `Function` under `is` (registry not threaded into `types::callable_kind_of`); an
incomplete `implement` surfaces at the *call* (not statically).

**M5.5b DONE (2026-08-27): the `extends` chain at runtime (S-61).** `ProtocolDef` gains
`extends: Option<u32>`, resolved at load (parent-first — the parent must already be a defined
protocol). Registry walks the chain: `transitively_declares` (requirements transitive),
candidacy by **direct** `implement` block with **ancestor subsumption** (a directly-implemented
ancestor is dropped when a directly-implemented descendant is present, so chain-related
protocols never make each other ambiguous — only genuinely *unrelated* protocols do),
**nearest-default-wins** (`nearest_default` walks self→parent→grandparent), and
`type_implements` is **transitive** (`x is Child ⇒ x is Parent`). Dispatcher-value `is
Procedure/Function` refined (registry threaded into `types::callable_kind_of`). Tests: 3-deep
chain resolves each member along the chain; nearest default wins + impl beats all; `is`
transitivity; an unimplemented inherited requirement raises `protocol-not-implemented` at the
call (**extends parent requirements enforced**); extends-of-undefined raises. `tests/protocols.rs`
now 15. All gates green (conformance 180, wasm32, hygiene 6/6).

- **★ Finding — the S-61 `extends` cycle rule is vacuous.** Parent-first load ordering means
  an `extends` target must already be defined; a forward/self reference reads an
  uninitialized cell → `used-before-defined`/`name-not-defined`. So an `extends` **cycle
  cannot be written** — the "cycle = static error" rule never fires. No cycle detection built
  (dead code). **Flagged to the user:** if cycles should be *constructible* + detected, that
  needs a different model (static protocol-name resolution with forward refs). [Test:
  `extends_of_an_undefined_protocol_raises`.]

**M5.5c DONE (2026-08-27): static conformance checks — M5.5 COMPLETE.** A resolver post-pass
(`resolve/walk/protocols.rs`) over same-module protocols/implements, run after `globals` are
collected (like `check_with_targets`). Five ratified static slugs (user 2026-08-27; rubric
App A): **`dispatch-parameter-default`** (a member's first param may not have a default),
**`protocol-signature-mismatch`** (an impl method's arity/block-param ≠ the member's; also a
child re-declaring an ancestor member with a non-conforming shape), **`implementation-parameter-default`**
(an impl restates a member's default), **`incomplete-implementation`** (an `implement` omits
required members of the chain — names each + the requiring protocol), **`not-a-protocol-member`**
(a method the protocol doesn't declare). Signature "shape" = ordinary-param count + block-param
presence (names are the impl's own). Cross-module scope: an imported `extends` parent / imported
protocol is invisible to the resolver, so its completeness checks (missing-member, not-a-member)
fall to load (`module-load-error`, per spec); the local checks still apply. Tests: 7 new static
cases (`resolve_diags` helper) + the two previously-runtime missing-member tests converted to
static; `tests/protocols.rs` now 22. Decision made & flagged: static (resolver) over load-time,
honoring the spec's "static error"; the S-61 cycle rule is vacuous (spec reworded `8a11c06`).
Deferred (non-gating): member-parameter defaults (edge; needs driven-default machinery).
All gates green (conformance 180, wasm32, hygiene 6/6).

**M5.4 DONE (2026-08-27): native modules — the full member set (a+b).** Host API:
`Registry::register_module(NativeModule)` + a builder (`.function`/`.constant`/`.foreign`/
`.record`), all four E§5.5 member kinds (functions, consts, foreign values, records — **no**
`parameter`/protocol/implement, per **S-44** ratified `fdd7473`). Native modules **pre-load**
into the module table at instance creation (ids `1..=k` in registration order — replay input,
MD §6) as synthetic modules: an empty-`Module` AST, member-name `globals` (so `m.member` +
wildcard resolve through the M5.2 machinery), and namespace cells of materialized values.
`import`ing one finds it via `by_path` (never suspends) and binds it like a Doodle module —
`m.member`, `import n.*`, cross-module call, `m.Point(...)` construction + `x is m.Point`, and
a foreign value with an exactly-once finalizer (E§4.5). Function members join the flat
intrinsics in one `CallableTarget::Intrinsic` id space, appended after the prelude
(`prelude_count` gates what seeds each module's namespace, so a native fn is bound only in its
own module). A native module the host didn't register isn't in `by_path` → the import suspends
→ host `NotFound` → `module-not-found` (E§13 "primitives absent ⇒ fail to load" — the existing
not-found path). S-32 is structural (registry consumed at load); `DuplicateModule` host error
on a name clash. Files split for length: `machine/native.rs`, `machine/intrinsic/ctx.rs`
(extracted `IntrinsicCtx` + `apply` out of `intrinsic/mod.rs`). Tests: `tests/native_modules.rs`
(8). All gates green (conformance 180, wasm32, hygiene 6/6).

**M5.3a DONE (2026-08-27): `exports` enforcement.** The resolver builds
`ResolvedModule.exports` (`None` = no `exports` statement = all public; union across multiple
statements; each exported name must be a declared global, else the static **`undeclared-export`**
`{module, name}`). Three cross-module membership sites — `m.member` (field_read), `import m.member`
(bind_member), `import m.*` (wildcard_lookup) — consult `ResolvedModule::member_visibility` →
`Membership::{Exported, Private, Absent}`. Private → **`not-exported`** `{module, member}` (loud-
and-true, points at `exports`); absent → **`no-such-member`** `{module, member}` (the module
container's access-miss kind — modules **no longer reuse `no-such-field`**, per user ruling); a
wildcard's private name surfaces `not-exported` on the error path. `exports` is a runtime no-op
(resolve-time surface only). Slugs ratified (user 2026-08-27; App C S-58 + rubric App A); also
fixed the stale rubric row `assign-to-undeclared` → `undeclared-assignment`. Native modules stay
all-public (`exports: None`; a native `exports` API is future). Files split: `machine/assign.rs`
(assignment scheduling out of `control.rs`, which was at 495 and would exceed the limit). Tests:
`tests/exports.rs` (7) + updated 3 module-miss assertions (`no-such-field` → `no-such-member`).
All gates green (conformance 180, wasm32, hygiene 6/6).

**M5.3b DONE (2026-08-27): the `module … end` file-level rename — M5.3 COMPLETE.** The resolver
unwraps a **sole file-wrapping** `module Name … end` (its body becomes the file module's top
level, its docstring the module's; `Name` is documentation — no runtime effect), and the machine
runs the wrapper's body in the module frame (a `Node::ModuleDecl` reaching the machine is always a
valid wrapper). Any `module` block that is **not** the sole top-level statement — alongside other
statements, or nested inside a wrapper — is the static **`nested-module`** (provisional slug,
retired when sub-namespace modules land, D-M5-5). Tests in `tests/exports.rs`. Spec note: L§11.1 + App D.1 updated by the spec
author (2026-08-27) — App C S-14 closed whole (module-block half + the M5.3a `exports` corners).

**M5.7 (real Stringable/Hashable dispatch) — DONE (2026-08-27; ★ D-M5-1 resolved).** The engine
natively registers `Stringable{to_string}` and `Hashable{hash}` at instance load, each with a
**native default** (renderer / structural hasher); the names `Stringable`/`Hashable`/`to_string`
seed every module's prelude (`hash` is not a bare name). **Interpolation** drives an explicit
`implement Stringable`'s `to_string` (a real, can-raise call — `protocol::enter_unary` +
`Cont::StrInterpRendered`), else the native renderer; the member is invoked by id (hidden binding,
S-37). **Dict keys** drive an explicit `implement Hashable`'s `hash` at insert/lookup/literal
(`dict::hash_plan` + `Cont::{DictBuildHashed,IndexReadHashed,IndexAssignHashed}`, which GC-root the
in-flight dict), else the native `check_hashable`+`hash_value`. `x is Stringable` is total; `x is
Hashable` = natively-hashable-or-explicit-impl. Decisions Q1 (compound/records → M9a) and Q2 (both
user-implementable now); riders: `print`/error-rendering stay native, `is` reflects native coverage
(all in D-M5-1). Files split for length (protocol/{mod,load,dispatch,dtype}, dict/{mod,index}). Tests
`tests/wellknown.rs` (16). All gates green (20 suites, conformance 180, wasm32, hygiene 6/6).
doodle-rust `<pending>`.

**Known gap (track with S-31 cross-module conformance):** a malformed `implement Stringable/Hashable
for T` (wrong arity, stray method) is **silently registered** — a native protocol is invisible to the
same-module resolver, the same gap as any cross-module `implement`, whose S-31 structural check routes
to a load-time `module-load-error` that is not yet built. Not a new defect (pre-existing category); no
memory/determinism impact. Discharge when cross-module `implement` conformance lands.

**M5.8 (prelude as a star-import) — DONE (2026-08-28).** The S-43 per-module seeding is retired: the
built-in type values, `Error`, the well-known protocols + `to_string`, and the flat intrinsics now live
in **one shared pre-loaded prelude module** (id after the natives, path `prelude`, registered `Loaded`).
`seed_namespace` binds only a module's own globals; every source module implicitly wildcard-imports the
prelude (`machine.prelude`, main set in `load_full`, imports in `import.rs`). Free-name resolution goes
through `control::lookup_free` (own namespace → wildcards); protocol `implement`/`extends`/type-name
resolution uses it too. Per the 2026-08-28 ruling (2f17007), the prelude is an **ordinary wildcard** (no
special tier): a prelude/user-wildcard **distinct-binding** collision is `ambiguous-import` at use
(naming "prelude"); `wildcard_lookup` now dedups hits **by cell identity** (M5.2c marked by name — this
is the S-13 refinement the ruling asked for), so same-cell aliases are one binding. Assignment to a
prelude name stays a static `undeclared-assignment`, so the shared prelude is read-only. No-observable-
change gate holds: full native suite + conformance (180) unchanged. Tests `tests/prelude.rs` (5). All
gates green (21 suites, conformance 180, wasm32, hygiene 6/6). doodle-rust `<pending>`.

**FOLLOW-UP (tracked, due before M6): D-M5-6 shadowing warning.** A declaration that hides a prelude
name (`let print = 5` … `print("hi")` → not-callable) must warn per L§5.1 (ratified 5c627fb; semantics
pinned, implementation slipped). Approach (no resolver-API change): a **post-resolve load-time pass**
diffs a module's `ResolvedModule.globals` against the prelude's export set and emits the L§5.1
shadowing warning at each colliding declaration's span (the prelude is registered before first load, so
its exports are known at load). User-wildcard shadowing is the linter's/import-time job (exports known
only at import execution) — the consistent end-state, not part of this follow-up. IDE (M6) is the consumer.

**M5.9 (turtle wrapper + 3-module cell-aliasing — the M5 acceptance headliner, exit #1) — DONE
(2026-08-28).** A native `turtle_native.pen` primitive, a Doodle `turtle` wrapper (`parameter
pen_color` + `to forward() pen(pen_color) end`), and a user module: `with pen_color = "red"` rebinds
the wrapper's own parameter cell (S-39 live alias), changing the color drawn inside the imported
`forward` — `black → red → black`. **Cross-module `with` (ratified 2026-08-28):** a `with` target now
resolves like any free name via `control::param_cell` (own namespace → wildcards, `wildcard_cell`
returning the exporter's cell), so an imported parameter is `with`-bindable through a **selective or
wildcard** import; two distinct wildcard bindings raise `ambiguous-import` (a `with` target is a use).
`param_cell` checks the resolved cell is a `parameter`, else the runtime **`with-target-not-parameter`**
— a NEW `ExceptionKind` sharing the **same slug** as the static `DiagnosticCode` (S-58 dual-catalog rule,
like `procedure-in-expression`; rubric App A updated). The resolver defers a free `with` target to
runtime only when a selective import or a wildcard could supply it (`has_wildcard_import` flag), else
still static (typo caught). Tests `tests/cross_module_with.rs` (6 fixtures — all the ratified cases).
All gates green (22 suites, conformance 180, wasm32, hygiene 6/6). doodle-rust `<pending>`. **Details
`{name, module, kind}` are `{}` today (the pre-M6 details pass populates them).**

**★ MILESTONE M5 — Modules, imports, protocols, prelude — COMPLETE (2026-08-28).** M5.10 closed it,
run as three landable sub-chunks:

- **M5.10a — conformance chapters — DONE.** Conformance runner gained **multi-module vectors**
  (directory-as-fixture: a dir with `main.doodle` + sibling `<name>.doodle`; `import name` →
  `<name>.doodle`, else `module-not-found`). Behavioral L§10 / L§11 / §4.12 fixtures; 180 → **191**;
  README documents the directory form (native-module/capability scenarios stay integration tests).
  doodle-rust `70f1908`.
- **M5.10b — determinism gate — DONE.** GC-stress determinism extended over module loading + dispatch,
  and (after the review) the actual M5.7/M5.9 driven paths (driven `hash`/`to_string`, cross-module
  `with`) under `collect_at_every_safe_point`. doodle-rust `bd016cc`.
- **M5.10c — exit review + close — DONE.** 4-lens adversarial read-only review (module-load×suspend/
  resume, cell-aliasing/GC, dispatch/ambiguity, prelude/name-resolution). Findings fixed + tested
  (doodle-rust `a0f0884`): qualified `P.member`/`member_signature` now walk the `extends` chain (was a
  panic on an inherited member); `drive::resolve()` on an import suspension faults instead of
  `unreachable!` (was a release panic reachable via the WASM facade); well-known-protocol `implement`
  blocks are now conformance-checked (a typo'd method was silently no-op'd) + an `enter_unary` arity
  backstop; `Registry::register` reserves the fixed prelude names. No CRITICAL/determinism/memory-safety
  findings — cell-aliasing, driven GC-rooting, load-state/cycle detection across suspend/resume, S-8, and
  prelude precedence all verified correct.

**App C discharged:** S-8/S-13/S-14/S-31/S-32/S-39/S-44 all landed (code + tests, each annotated
`[Code: M5.x]` in implementation.md App C). Accept clauses: #1 (cell-aliasing) `cross_module_with.rs`;
#2 (assign-to-import static) `resolve.rs`; #3 (wildcard ambiguity) `prelude.rs`/conformance; #4 (failed
module) — failed-state proven, the re-import **re-raise is the ratified M9b deferral** (S-8: reload is
environment-level); #5 (missing platform primitive) covered generically by `module-not-found`; #6 (M4
carve-outs) `wellknown.rs`/`protocols.rs`. wasm = the build gate (Node-executed wasm tests deferred per
AD8). All gates green (22 native suites, conformance 191, wasm32, hygiene 6/6).

**Open M5-adjacent follow-ups (tracked, not blocking):** (a) the **D-M5-6 shadowing warning** —
**DONE (M6.0a, doodle-rust `b3a9eb3`)**: it needed a load-time channel, so the user pinned **S-63** (the
instance load-diagnostics record; also closes M1.1's three discovered deltas — warnings channel, schema,
ordering) and the code built it (`load::prelude_shadowing` feeding `Machine.load_diagnostics`, read via
`Instance::load_diagnostics(since)`); (b) the pre-M6
**`details` population** for the runtime `Error` kinds — **DONE (M6.0b, doodle-rust `af1fa38` + `addaa9d`)**:
every kind carries its S-58 schema (a few optional sub-fields deferred, noted in code + rubric); (c) a minor
diagnostic-precision limit — `with <prelude-const>` in an import-less module says "none is declared"
rather than "is a constant" (sound per D-M5-3; sharpening needs the resolver to know prelude names,
D-M5-6 territory). None affect a running program.

**★ MILESTONE M6 — Full observation surface + IDE debugger — STARTED (2026-08-29).** Working plan
written: **`plan/plan-m6.md`** (M6.0 ratified pre-M6 obligations → M6.1 value inspection / M6.2 rich
frames → M6.3 host pause / M6.4 breakpoints / M6.5 raise-trap → M6.6 stepping+observation-mode / M6.7
aux-eval → M6.8 drive-script conformance → M6.9 full IDE debugger → M6.10 close). **The fact that
dominates M6:** the drive scaffolding already exists — `Directive` (all 6), `PauseReason`
(`Breakpoint`/`RaiseTrap`/`HostPause`/`SliceEnd`), `BreakpointId`, depth-anchored `should_pause` for
`Step*` — as **unwired shells** (`Continue` doesn't pause, no breakpoint index, no host-pause flag, no
raise-trap). So M6 is fill-in-the-shells + complete the E§8 pull surface (record/dict inspection, frame
locals + `with` bindings, callable reflection, aux-eval — today only scalars+lists are inspectable) +
the browser debugger. Risk is **breadth, not depth**; the two tricky mechanisms are **raise-trap**
(paused-mid-raise state) and **aux-eval** (nested `to_string` on a paused instance). **Three decisions
RESOLVED (user, 2026-08-29):** **D-M6-1** fine observation mode → **S-62** (spec landed `8eed2e3`: fine
safe points = completion of every non-leaf subexpression, by syntactic form; leaves excluded;
observation-only — GC/limits/cancel/budget/fuel stay at statement safe points; part of replay identity);
**D-M6-2** → **portable drive-script conformance format now** (not deferred to M8); **D-M6-3** → **fuller
debugger in-gate** (watch-it-run, elided-history viz, expandable value trees, raise-trap UI). **D-M6-4**
stated-not-asked: a minimal engine-level callable-reflection API in M6, Doodle `help` stdlib stays M9a.
Out of scope: live edit (E§8.9), conditional breakpoints (aux-eval is their future foundation).

- **M6.0a DONE (2026-08-30, doodle-rust `b3a9eb3`): the S-63 load-diagnostics record + D-M5-6 warning.**
  A successful load may produce warnings — a module global hiding a prelude name (built-in type, `Error`,
  a well-known protocol, or a host intrinsic) shadows it (L§5.1). E§3.2 had no channel for warnings on a
  successful load, so the user pinned **S-63** (spec `4be2fa7`): one instance-scoped, monotonic
  **load-diagnostics record** appending every front-end diagnostic for every module loaded or attempted
  (entry module at load; imports as they load mid-drive; errors included), read by pull —
  `Instance::load_diagnostics(since)` — on a `Ready`/stopped instance. Errors keep their control-flow
  channels (`LoadError`; a broken import's `module-load-error`); the record is the one display surface.
  Code: `Machine.load_diagnostics`; `load::prelude_shadowing` (globals∩prelude-names diff, no
  resolver-API change) feeding the entry module at `load_full` and each import in `import.rs` (which also
  now accumulates imports' front-end diagnostics — previously dropped); deterministic order (load order,
  then span order), replay-stable, engine-owned/not-program-data. **Closes the three M1.1 discovered
  deltas** (warnings channel, diagnostic schema, diagnostic ordering) and D-M5-6's channel question.
  `machine.rs` crossed the 500-line soft limit with the new field, so its six `#[cfg(test)]` `Instance`
  helpers moved to the length-exempt `machine/tests.rs`. Tests `tests/load_diagnostics.rs` (7). Gates:
  native 519, conformance 191, wasm32 build, hygiene 6/6.

- **M6.0b DONE (2026-08-30): `Error.details` populated for every kind (rubric pin (b)).** The raise
  carries a structured `(key, DetailVal)` list; `make_error` builds the dict; `value_type_name` gives
  S-37 display type names. **Part i (doodle-rust `af1fa38`):** every non-argument kind — the ratified Q1
  schemas (type-mismatch `{operator, expected, got}`, undefined-ordering `{operator, left, right, nan?}`,
  not-callable `{type}`, unhashable-key `{type}`, …) + the `{}` thin kinds; split `control.rs` →
  `control/names.rs`. **Part ii (doodle-rust `addaa9d`):** the **`argument-error` split** (user S-58
  catalog `5fd99df`) into `missing-argument`/`unknown-keyword`/`duplicate-argument`/`too-many-arguments`
  — one fact per slug, `details` carrying data; each binding site (call/record/dispatch/block/intrinsic)
  picks the kind + fills details (`callee` threaded where named); split `call.rs` → `call/frame.rs`. Pin
  (a) proven by conformance `details-001..008` (read `e.details["…"]`, never the message). Rubric App A
  column updated. Gates: native 519, conformance 199, wasm32, hygiene 6/6. **Deferred optional sub-fields**
  (noted in code + rubric): with-target `module`, unhashable `field`, boolean-context type-mismatch `got`,
  not-a-protocol `got`, module-load-error per-diagnostic notes/suggestion.

- **M6.1 DONE (2026-08-30, doodle-rust `9be1b18`): structural value inspection (E§4.4/§8.4).** The pure,
  Doodle-code-free reads the debugger's value tree + stack panel render from — completing E§4.4/§8.4
  beyond scalars+lists. New `machine/inspect.rs`: records (`record_type_name`/`record_length`/
  `record_field_name`/`record_field`), dicts (`dict_length`/`dict_key`/`dict_value`, insertion order),
  callable reflection (D-M6-4 minimal: `callable_name`/`callable_is_function` (S-37 Procedure/Function)/
  `callable_position`/`callable_docstring`), `type_name`, `module_member_names`. Handle-minting readers
  host-owned (list_get discipline); wrong-kind is a `ValueError`, never a panic. Native 520, conformance
  199, wasm32, hygiene 6/6.

- **M6.2 DONE (2026-08-30, doodle-rust `d27dadf`): rich frame observation (E§8.2/§8.3).** Extends the
  frame surface beyond callable/call_site/tail_count: `Instance::frame_locals(index)` (a frame's
  params + `let`/`const` names→value, via `CallableInfo.slot_names` + the frame's slots; `None` = TDZ
  slot; callable + module-top frames, a block reports none), `Instance::frame_dynamic_bindings(index)`
  (the `with` bindings the frame opened — its `dyn_stack` range, cell→name via reverse namespace scan,
  current value), `Instance::tail_elided_history()` (the ring buffer of tail-overwritten callers,
  most-recent-first: callable handle + decl span). New `Binding`/`ElidedFrameObservation`; handle-minting
  host-owned. `Frame.dyn_depth` now read (dead_code allow dropped). Native 521, conformance 199, wasm32,
  hygiene 6/6.

- **M6.8 DONE (2026-08-31, doodle-rust `7621c84`): drive-script conformance runner (D-M6-2, §4.3).**
  A new `mode: drive` in the existing `#!`-header conformance runner (Option 1, ratified: reuses
  discovery + is cheap for M7's C harness to re-parse; colocated with the program so lines can't drift).
  Setup (`break:`/`raise-trap:`/`obs:`) then ordered `do:`/`expect:`/`stack:` steps →
  `tools/conformance-runner/src/drivescript.rs` (parser) + `.../drive.rs` (executor): load → apply setup
  → drive each action (imports resolve transparently like `run` mode) → **full-transcript** compare
  (every step's outcome kind/reason/position/stack, in order — no spot-checks). Per the ratified riders:
  stack elements `L` / `name@L` / `name@L×N` (matcher checks only what's pinned; tail counts assert E§8.3
  elision); unknown `do:`/`expect:`/`stack:` tokens and the **reserved** `local:`/`render:` slots are
  parse **errors** (never silently skipped); a `suspended` stop asserts the request identity; a terminal
  `do:` is a clean fixture error, not an engine-assert panic. Grammar pinned **normatively** in
  `conformance/README.md` (Rust runner = reference parser; versioned via `mode: drive` → future `drive2`).
  Fixtures `conformance/v0.1/eng/E8.*` (6): directive matrix (breakpoint+raise-trap — `Continue` pauses at
  each, `run` runs through), per-iteration breakpoint refire, raise-trap pre-unwind, tail-`StepOver`
  constant-depth (`go@L×N`), fine-mode subexpression stepping (S-62). doodle-core untouched. Runner tests
  31, conformance 205 (199 lang + 6 drive), native 548, wasm32, hygiene 6/6. Deferred (reserved slots):
  capability-resolution steps (M7) + value/inspection assertions. **Next: M6.9** (IDE debugger, doodle-web).

- **M6.9a DONE (2026-09-01, doodle-rust + doodle-web): wasm debug bindings + drive-through-wasm parity
  gate.** M6.9 split into two sub-chunks (ratified 2026-08-31): **6.9a** the wasm/JS debug surface +
  parity gate, **6.9b** the CodeMirror UI. Bridged the whole E§8 surface to JS: a **directive** on
  `drive` (`run`/`continue`/`step`/`into`/`over`/`out`; `resolve` resumes under the remembered directive);
  breakpoints, raise-trap, observation-mode setup; the stack walk + **lazy** per-frame bindings + a pause
  **generation** token; `completedSpan`/`trappedRaise`/`trappedRaiseSpan`; `evalToString`; flat inspection
  readers (record/dict/list/callable/type/module). **Marshaling (ratified D-M6-3 rider):** structured
  reads cross as **plain GC-owned JS objects** via `js_sys` (no serializer); a frame needs no `.free()`
  (callable reflected to data, binding **names** eager, **values** lazy via `frameLocal(gen,i,slot)` — the
  only debug reads that mint a handle); the generation token makes a stale post-resume frame read a clean
  error; tail-elided frames ride the same `stackWalk` array with `elided:true`. doodle-core `observe.rs`
  gained the lazy split (`frame_local_names`/`_value`, `frame_dynamic_names`/`_value`; batch methods build
  on them). `js-sys` added as a doodle-wasm dep, pinned `0.3.103` to hold wasm-bindgen at 0.2.126.
  `@doodle-lang/engine` gained `debug.ts` (TS contract). **Gate:** new `engine/test/drive.test.mjs` ports
  the reference drive-script parser/executor and runs all six `E8.*` `mode: drive` fixtures **through the
  wasm surface** (same transcript as native — cross-surface determinism for the debug bindings). Native
  facade tests cover breakpoints→locals→inspection, raise-trap, step, aux-eval, generation staleness.
  Gates: native workspace green, native conformance 205/0, wasm32 clean, hygiene 6/6, wasm ship-size 243 KB
  brotli (<300 KB), doodle-web engine 124 pass / **5 skip** + typecheck. **Fixed in passing:**
  `conformance.test.mjs` now **skips** the M5.10a multi-module `import` run fixtures through wasm (imports
  are deferred M5-web work, E§6 — the facade faults `import-unsupported`); this was latent because the
  doodle-web CI had not run since M5.10a. **DISCOVERED (raise before 6.9b): module globals have no
  observation accessor** — `frame_locals` covers `to`/`fn` frames only; module-level `let`/`const` are
  globals (module `namespace`), so a **top-level** program's Locals panel would be empty. Central to the
  demo (most kid code is top-level). Options: add `module_global_names` + lazy value to the engine, or
  surface via a Module handle + `module_member_names`, or defer. **Next: raise the globals decision, then
  M6.9b** (CodeMirror debugger UI + debug-session driver + Playwright e2e).

- **M6.9b DONE (2026-09-01, doodle-rust + doodle-web): the IDE debugger (D-M6-3).** Globals decision
  ratified as **S-64** (Option A, landed 2026-09-01). Then: one more engine primitive —
  `resultHandle`/`currentResult` (the fine-stop value, S-62 watch companion) — and the **debugger UI**
  (`packages/demo/src/debug/`). Ratified **Debug ▸** model: Run animates to completion; Debug opts into
  breakpoints + stepping + panels. Pieces: a CodeMirror **breakpoint gutter** (click-to-toggle,
  `lineMarkerChange` so a state-only toggle re-renders — the e2e-only bug that a doc-less transaction
  didn't refresh the dot); a DOM-free **DebugSession** driver (drives a directive in fuel slices, fulfils
  turtle capabilities via the pump's now-exported `decodeValue`/`encodeValue`, stops at
  breakpoints/steps/raise-traps); a **DebugController** (lifecycle); **DebugPanels** (call stack with tail
  badges + elided history, the selected frame's Variables = locals + `with` + module globals as
  **expandable value trees**, a **raise-trap** pre-unwind readout, the **watch-it-run** value); a bounded
  handle-safe **value-tree** materializer (mints child handles + releases them, so the DOM holds no live
  handles). **Gates:** demo Node tests 8/0 (breakpoint→pause→globals→step→resume, raise-trap,
  watch-it-run, record tree), Playwright e2e 8/0 (gutter breakpoint → Debug pause → variables → Stop;
  Debug-no-breakpoint → done; step advances the line), typecheck clean, posture / first-load (431 KB <
  461 KB) / wasm-size green; doodle-rust native + wasm32 + hygiene 6/6. Accept criteria met end-to-end in
  the browser. **M6.9 COMPLETE.**

- **M6.10 DONE (2026-09-01): exit review + close. M6 COMPLETE.**
  - **Globals-ordering fix — DONE (doodle-rust `aa4661d`, doodle-web `49f9e2f`).** `module_global_names`
    returns each global's **`decl_span`** (E§8.2); the demo separates its own top-level globals (decl
    after the prelude) from the prepended library's (decl before → `userLineOf` null), user's under
    "Module globals", library's in a collapsed group.
  - **Multi-lens adversarial review (4 read-only subagents) — 2 confirmed defects, both FIXED
    (doodle-rust `96b7add`):** a **CRITICAL** aux-eval GC use-after-free (the nested drive moved the
    outer `unwind`/`reg`/`pending` into Rust locals, which `gc::collect` no longer rooted → a GC during
    a `Stringable` render freed the trapped raise value; fixed by rooting them in `foreign_roots`; a
    regression test panics on a freed slot without the fix) and a **MAJOR** dynamic-parameter leak (the
    aux fault path truncated `dyn_stack` instead of the `with` cell writeback; fixed with
    `unwind::restore`). Minors fixed: `gc_threshold` restore, cancel→`Cancelled`, `frame_dynamic_*`
    fail-soft slice, `trapped_raise` doc, gen-wrap note. Handle-discipline + breakpoint/fine-mode lenses
    found no correctness/determinism bugs (all invariants verified sound).
  - **Determinism-gate extension — DONE.** `the_observation_surface_does_not_perturb_the_program_trace`:
    straight vs stepped/fine/breakpointed + full pull reads, under a collect at every safe point →
    identical output + outcome (E§7.7/§11).
  - **App C discharged:** S-18/S-21/S-22 (two review fixes recorded on S-22), S-34 marked; S-15/S-16
    `NestedSuspend` consistency re-verified.
  - **Debugger UX pass (feedback) — DONE (doodle-web `b5128a2`):** panels fixed-size + always-visible
    while debugging; an "under" highlight colour for a pause inside a call whose source isn't shown;
    animated `right`/`left` turns; a speed slider (animation speed + paced `continue` with per-line
    highlight). Gates all green; CI verified.

**★ MILESTONE M6 — Full observation surface + IDE debugger — COMPLETE (2026-09-01).** All exit criteria
met; the deployed browser demo carries the full debugger (breakpoints, stepping, call-stack/variables/
value-tree/raise-trap/watch panels). **Next milestone: M7** (per implementation-plan §5 / M7 paragraph).

- **S-65 DONE (2026-09-01): per-operation result cap — the "latency rail" (E§10.2).** Removing the demo's
  "huge `**` can freeze the page" disclaimer, investigation showed R8's guard fixed the *example*
  (`9 ** 9 ** 9` faults on heap in ~2ms) but not the *class*: a bignum `**`/`*`/repetition whose result
  fits the heap but is expensive to compute still froze the tab (`1500000 ** 1500000` = 2.3s, Stop can't
  interrupt an atomic op, S-40). Fix (ratified): a **third `Limits` rail** — `max_op_result_bytes` — a
  distinct `LimitExceeded(OpResult)` faulted **before** computing at every result-growing admission point
  (`admit_op_result`, renamed from `admit_bignum`, now covering string repeat), from the same cheap size
  estimate. Default = heap-bounded (no change for other hosts); the demo sets 1 MiB → the pathological
  ops now fault in ~ms. Rejected again: a superlinear compute charge (couples replay to the multiply
  algorithm) and interior safe points (relitigates S-40). Spec: E§10.2 (three rails: space/work/latency);
  App C S-65. Disclaimer removed; a friendlier "that computation is too big" status added. Native +
  wasm32 + hygiene 6/6, doodle-web all green.

- **M6.7 DONE (2026-08-31, doodle-rust `0c328fc`): auxiliary evaluation (E§8.4, S-22; riders `39fa1e8`
  + `bc5972b`).** `Instance::eval_to_string(handle, fuel) -> AuxOutcome { Rendered(Handle) | Raised(Handle)
  | Faulted(EngineFault) }` in new `machine/aux_eval.rs`. A type with an explicit `implement Stringable`
  drives its `to_string` in a **nested drive** (mirrors the reentrant block-consumer loop — `step::step`,
  not the debug drive loop, which is what suppresses breakpoints + raise-trap); no explicit Stringable →
  the pure native seam (`stringify::render`), no drive. **Saved/restored debug context**: register,
  in-flight unwind (cleared — load-bearing at a raise-trap pause where the outer raise is armed), program
  budget/fuel, `fine_span`/`safe_point_stmt`/`directive`, pending, tail-ring, and stack heights
  (frames/dyn/handling/foreign, truncated back). The aux drive runs on its **own per-call `fuel`** (a
  swapped-in `FusedCounter`), so it never charges the program's budget; exhaustion faults it (one-shot).
  **Effects persist** (ratified `39fa1e8`: aux eval is effectful — only the debug context restores);
  suspension inside → nested-suspend fault (S-15); a **non-String `to_string` result raises the same
  `type-mismatch`** interpolation does (`bc5972b`). Tests: native-scalar render, explicit-Stringable
  drive, pause-intact (position/depth unchanged + resume completes), raising `to_string`, runaway →
  own-budget fault, at-a-raise-trap-pause (armed unwind saved/restored), non-String → type-mismatch.
  Native 548, conformance 199, wasm32, hygiene 6/6. **Next: M6.8** (drive-script conformance runner, D-M6-2).

- **M6.6 DONE (2026-08-31, doodle-rust `e895db2`): stepping refinement + observation mode (E§7.4/§8.8,
  S-62; E§8.8 one-axis `aac6766`).** `ObservationMode { Statement, Subexpression }` on `Config` (kept
  `Copy`) + `Instance::set_observation_mode`/`observation_mode`; `create*` threads it. The eager/lazy
  local-capture axis was **removed from the spec** (pull inspection has no capture step). **Fine safe
  points reuse the existing `step → Some(depth)` channel** — no `SafePoint` enum, no drive-loop change: in
  `Subexpression` mode `step` emits `Some(depth)` at each non-leaf subexpression completion **without
  accounting** (no `limits::safe_point`/`poll_cancel`), so budget/fuel/GC/cancel stay at statement points
  and a fault lands at the same instant in both modes (S-62). `breakpoint_hit` returns `None` mid-statement
  (no false breakpoints at fine points); `RunToCompletion` ignores `Step`. Detection: a
  `fine_completion_span` predicate over the popped cont (`BinApply`/`UnaryApply`/`AssertBool`/`FieldRead`/
  `IndexApply`/`IndexReadHashed`/`StrInterp`/`StrInterpRendered`); an `and`/`or` short-circuit records its
  span in `logical_rhs`. Value at a fine stop = `result()`; position = `completed_position()` (`None` at
  statement stops). `if`-expression branch results land at the branch's final statement safe point (arms
  are blocks) — no separate fine point; list/dict literals are outside the fine set (E§7.4). Tail-aware
  `StepOver` needed no code change — a test verifies constant depth + frame serial across tail reuse.
  `step.rs` crossed the 500-line soft limit; split the `dispatch` match into `machine/step/dispatch.rs`.
  Tests: operator/field+index/if-expr/interpolation fine traces, leaves-not-stopped, coarse-has-none +
  runtime switch, same-fault-instant under a budget, fine-trace determinism double-run, tail-StepOver.
  Native 541, conformance 199, wasm32, hygiene 6/6. **Next: M6.7** (auxiliary evaluation, S-22 — the
  nested-`to_string`-on-a-paused-instance mechanism).

- **M6.5 DONE (2026-08-31, doodle-rust `e5b23db`): raise-trap (E§8.7, S-18; E§8.7 sharpened `6adb616`).**
  Pauses `Paused(RaiseTrap)` at each raise **before the stack unwinds**, so the debugger sees the raising
  frame intact; resuming continues the unwind. No separate paused-mid-raise *state* was needed — the pending
  raise already lives in `Unwind::Raise` (armed, not yet stepped), so the mechanism is a one-shot
  `trapped: bool` on that variant + a drive-loop check **before** `step` (where the unwind runs). All raises
  funnel through one arming chokepoint (`arm_raise` for program/engine, `arm_raise_value` for foreign
  `resolve(Raise)`), so S-18's unification is structural: `Instance::take_raise_trap()` sets `trapped` and
  returns true once per armed raise; the resumed drive sees it set and steps into the unwind unchanged. New
  `machine/raise_trap.rs`: `Machine.raise_trap_enabled` (off by default) + `set_raise_trapping(bool)` /
  `raise_trapping()` / `trapped_raise() -> Option<Handle>` (raised value) / `trapped_raise_position() ->
  Option<Position>` (raise site from the in-flight trace); `PauseReason::RaiseTrap` now wired. **Directive-
  gated** (ratified): fires under `Continue`/`Step*`, `RunToCompletion` ignores it (E§7.3's outcome list
  already excluded `Paused(RaiseTrap)`; §8.7's silence was the gap). Tests: pre-unwind stack intact (a `with`
  binding still live at the trap, restored only on resume), trap-fires-even-when-caught, engine-raise
  unified, off-by-default, RunToCompletion-ignores. `machine.rs` crossed the 500-line soft limit; split the
  cancel+pause host-control `impl Instance` methods into `machine/controls.rs`. Native 533, conformance 199,
  wasm32, hygiene 6/6. **Next: M6.6** (stepping refinement + observation mode, S-62 fine safe points).

- **M6.4 DONE (2026-08-30, doodle-rust `94fa7ef`): breakpoints (E§8.6, S-21 ratified `45e1bca`).**
  Addressed by the host-owned **canonical id** (not an engine module index). New `machine/breakpoint.rs`:
  `Breakpoints` on the `Machine`, `Instance::{set_breakpoint(canonical, line) -> BreakpointId,
  clear_breakpoint(id), breakpoints() -> [BreakpointInfo{id, canonical_id, line, resolved}]}`. The entry
  module now carries a canonical id (E§3.2): `load` threads a `module_path` (default `"main"`, override
  `Instance::create_with_module_path`), seeded into `by_canonical`. Resolution: the resolver's existing
  `stmt_spans` + a new **line index on `Ast`** (`line_starts`, parser-built; omitted from `Ast`'s Debug so
  golden snapshots are unchanged) → `resolve_line` snaps forward to the first statement at/after the line
  (`min_by_key(line, span.start)` ⇒ first-on-line + code-less-line snap). **Unknown/unloaded canonical or
  past-EOF line = pending, not an error**; `reresolve_breakpoints` re-snaps at every source-module load
  (import.rs) — the set-then-run + reload rule. Runtime match is by the **statement node about to run**
  (`machine.safe_point_stmt`, recorded in `step`, gated to the outer drive via `reentry_depth == 0`),
  checked in the drive loop under `Continue`/`Step*` (never `RunToCompletion`), before the Step decision —
  so a loop-body breakpoint refires each iteration. `ast.rs` crossed the 500-line soft limit, split
  `ast/arena.rs` out. Native 529, conformance 199, wasm32, hygiene 6/6. **Known gaps (noted):** (a) a
  breakpoint inside a native-invoked (reentrant) block does not fire — reentrant drives aren't
  pausable/resumable (same E§5.4 limitation as capability-suspend-in-native-consumer; M7 foreign-yield);
  (b) the intrinsics-carrying load path (wasm facade) defaults the entry canonical to `"main"` — the real
  filename is wired in M6.9. **Next: M6.5** (raise-trap, E§8.7/S-18 — the paused-mid-raise mechanism).

- **M6.3 DONE (2026-08-30, doodle-rust `7dd490f`): host-requested pause (E§8.8).** `PauseToken` (new
  `machine/pause.rs`, sibling of `CancelToken`) over `Machine.host_pause: Arc<AtomicBool>`;
  `Instance::pause_token()` hands out thread-safe clones. Unlike cancel, a pause is **resumable, not a
  fault**, and **one-shot**: the drive loop consumes it at the next safe point via
  `Instance::take_host_pause()` (atomic read-and-clear), gated on `safe_point.is_some()` so it never fires
  mid-expression, and returns `Paused(HostPause)` with state intact — checked **before** `should_pause` so
  it stops **regardless of directive** (E§8.8: a host control, not a `Step*` decision) and wins over a
  Step pause reason. No unwind armed. A pause requested while suspended fires on the resumed drive's next
  safe point. Tests: stops+resumes under both `RunToCompletion` and `Continue`; mid-drive request preempts
  the Step reason; one-shot. Native 523, conformance 199, wasm32, hygiene 6/6. **Next: M6.4** (breakpoints,
  S-21 — the first real `Continue` content), then M6.5 (raise-trap).

**Milestone M3 — WASM binding + first public demo — COMPLETE (2026-08-23; M3.1–
M3.9 all landed + exit-reviewed; demo live at
https://doodle-lang.github.io/doodle-web/).** The record below stays as the M3
history.

**Milestone M2b — Drive layer — COMPLETE (2026-08-03; M2b.1–M2b.7 all landed
+ exit-reviewed).** The host/embedding layer ships: the resumable drive-state
machine, boundary value model, intrinsic foreign functions + suspending
capabilities, reentrant drives + native block-consumers + S-46 non-local
exits, foreign values + finalizers, cancellation, and the minimal observation
surface. The per-item detail below stays as the M2b record.

**Milestone M3 — WASM binding + first public demo — COMPLETE (2026-08-23).**
Working plan **`plan/plan-m3.md`** written + 5-lens reviewed + landed (2026-08-03;
decomposed M3.1–M3.9: fuel/SliceEnd, turtle surface + native block-consumer,
the S-15 nested-drive-suspend prototype, the wasm facade + size gate, the JS
fuel pump + conformance-through-wasm, turtle rendering + S-23, the demo page,
deploy, exit review). Ten environment/product decisions surfaced; three
forks (**#2 S-15 resolution, #6 turtle registration, #7 turtle vocab**) await
the user. **#6/#7 RESOLVED (user, 2026-08-20)** and shipped in M3.2 (turtle =
Doodle code over three host primitives; all-capabilities). **#2 (S-15)
RULED (user, 2026-08-20): forbid-and-fault baseline, no second prototype**
— fault not raise (the S-46 parity stance extended to suspension); the M7
C-ABI-yield path characterized in E as the compatible extension; full
riders on plan-m3 §Decisions #2. **M3.3 DONE** (see below). **Decision #1
(js/web placement, AD7) RULED (user, 2026-08-21): a new `doodle-web`
submodule for all JS/TS + web; `doodle-rust` stays pure Rust** (the
`doodle-wasm` crate stays; `doodle-web` consumes its wasm-bindgen artifact).
The conformance-through-wasm determinism gate spans repos → a
workspace-superrepo CI job. `doodle-web` gets created at **M3.5** (M3.4 is
`doodle-rust`-internal). Remaining M3 forks: #3 hosting, #4 privacy, #5 npm,
#8 print surface, #9 R8 guard, #10 budget (all resolvable when their item
lands).
- [x] **M3.1 — bounded-run fuel + `Paused(SliceEnd)` (S-40). DONE**
      (doodle-rust `a7b3963`; **E§7.2/§7.3 + App C S-40 pinned**, user-ruled
      2026-08-03). `run_slice`/`resolve_slice(fuel: Option<u64>)` variants
      (chosen over a param on `run`/`resolve` — distinct forms avoid a C-ABI
      unbounded sentinel and make the outcome contract signature-level); a
      per-call fuel bound fused with the lifetime step budget (whichever hits
      0 first stops: budget→terminal `LimitExceeded`, fuel→resumable
      `SliceEnd`; the fault wins a same-instant race); the slice does not gate
      execution or perturb GC (E§7.7). Riders pinned: fuel orthogonal to the
      directive; per-call, never banked (`fuel=0` = immediate SliceEnd); a
      control signal at a terminal transition defers to the terminal outcome
      (jointly with the §10.1 cancel pin); program-invisible, outside replay
      identity. 4-lens review: determinism CLEAN, 3 findings folded
      (completion wins the exact-fuel boundary; slice `Option<u64>`, no
      sentinel; fuel=0).
- [x] **M3.2 — platform primitives + the Doodle turtle library. DONE**
      (doodle-rust `aa6425a` turtle + `13fbdef` the `**` sweep; **E§11 amended**
      for transcendental determinism). The turtle is ordinary Doodle code over
      three platform primitives — `draw_line`/`set_turtle`/`clear_canvas`, **all
      suspending `to` capabilities** (user-ruled all-capabilities 2026-08-20:
      uniform, no new engine machinery, engine stays turtle-agnostic) — plus
      provisional `sin`/`cos` natives via the bundled deterministic `libm`
      (default-features=false soft-float, so trig is bit-identical native↔wasm).
      `doodle/turtle.doodle` holds all state/geometry/color; colors are named +
      `0xrrggbb` hex-int + RGB(A) channels. 4-lens read-only review: 9 findings
      folded. **Headline (MAJOR, found+fixed): the `**` float path called
      platform `f64::powf`** — a transcendental determinism leak the sin/cos
      sweep missed and the new E§11 clause forbids; now `libm::pow`, with an
      exact-bit golden. Also folded: alpha-clobber on an unknown color name
      (fixed + regression test); left/pendown/showturtle/home coverage; doc
      fixes; plan-doc reconciliation.
      - **Deferred (approved, tracked):** the `#rrggbb` **string** color form
        waits on string-decomposition primitives (grapheme/codepoint access)
        the runnable subset lacks — add to `pencolor` when string ops land
        (~M4/stdlib). The `0xrrggbb` hex-int + named + channel forms ship now.
      - **M5 (real module system) will fix:** the single-module prepend means
        library globals (`forward`, `right`, `home`, `repeat`, `turtle_*`, …)
        share the user's namespace, so a user redeclaration is a duplicate-name
        error; and `pencolor(r, g)` (arity slip) forwards a `nil` channel with
        no validation (no exceptions until M4). Both are provisional-prepend
        limitations, not bugs.
      - **File-length split: DONE** (doodle-rust `2fd3423`). The two files
        M3.2 pushed over the 500-line soft limit are split along natural
        boundaries: `arith.rs` (519→281) moved its inline test module to a
        sibling `arith/tests.rs`; `machine.rs` (584→437) moved the
        instance-construction family (`create`/`load*`/`load_full`) to
        `machine/load.rs` as its own `impl Instance` block. Pure refactor,
        public paths unchanged; hygiene now green with zero warnings.
- [x] **M3.3 — S-15 nested-drive-suspend: forbid-and-fault. DONE**
      (doodle-rust `4a3418f`; **E§5.4/§7.6/§7.2 + App C S-15 + MD §14/§19**).
      Decision #2 (user-ruled 2026-08-20, re-confirmed 2026-08-21): a suspending
      capability reached inside a **native** block-consumer's reentrant drive is a
      new, distinct **`Faulted(NestedSuspend)`** — terminal + deterministic, the
      M2b.5a `Internal` stub made deliberate. The nested drive runs on the Rust
      stack, so the native consumer's progress can't be frozen/resumed; a **Doodle**
      block-consumer (whose block runs on the engine's own stack) suspends
      normally — the turtle demo's `repeat` relies on that. "Suspend-the-outer-
      drive" is the same save/resume protocol a C foreign fn needs, so it is
      **characterized in E as the deferred M7 C-ABI-yield extension, not built**
      (mechanism analysis was conclusive; R4's "prototype both" satisfied by
      analysis). Tests: native-`each`-block-suspend → `NestedSuspend`; the parity
      Doodle-consumer-suspends-normally case. **Process note:** this was re-asked
      because the post-compaction summary dropped the 2026-08-20 ruling (recorded
      in discussions `778772a`); the user re-confirmed identically — no rework.
- [x] **M3.4 — `doodle-wasm` facade + binding size gate. DONE**
      (doodle-rust `e0dd50d`). The E§3–§8 surface via wasm-bindgen: a
      natively-testable `facade::Session` core + a thin `DoodleInstance`/
      `DriveResult` shell (Decision #1 keeps `doodle-rust` pure Rust). `demo`
      (print-only, conformance parity) + `turtle` configs; drive/resolve over
      fuel slices; opaque handle-`u64` boundary; string-tagged outcomes;
      output/currentSpan/cancel. **Size gate now binding** — `wasm-size.sh`
      measures the real `_bg.wasm` and CI installs a Cargo.lock-matched
      `wasm-bindgen-cli`. **178 KB brotli / 300 KB — 40% headroom; AD4's Unicode
      tables do not breach it (no §6.5 ladder / Decision #10).** 3-lens review:
      encoding/handle-discipline/gate correct; 6 nits folded. **Node smoke
      deferred to M3.5/doodle-web** (Decision #1 — no JS harness here). CI cost:
      the wasm-size job now `cargo install`s wasm-bindgen-cli (~2–4 min; can be
      sped up with a prebuilt binary later).
- [x] **M3.5 — `@doodle-lang/engine` pump + conformance-through-wasm. DONE**
      (doodle-web `042b35e` pump + `9f77ed3` conformance/CI; workspace submodule
      `8507e0a`).
      **Created the `doodle-web` sibling submodule** (public, npm-workspaces) with
      **`@doodle-lang/engine`** (Decision #5 RULED 2026-08-21). `build-wasm.sh`
      builds `../doodle-rust`'s wasm via wasm-bindgen (`--target web`); the
      **fuel-sliced pump** drives in ~8 ms slices via an injectable scheduler,
      capabilities→Promises (handler-throw → Doodle raise), stop checked between
      slices AND after each capability, position per slice, streamed output. 8 Node
      tests through real wasm; TS strict. 3-lens review, 10/11 folded incl. a
      **MAJOR** stop-button fix (was inert for capability-suspending loops — the
      animated-turtle shape). **Env note:** `NODE_OPTIONS` is broken in this shell
      (stale preload) — prefix Node/npm with `NODE_OPTIONS=`.
      - **Deferred (tracked):** O(n²) per-slice `output()` copy — add an
        output-since-offset method to the doodle-wasm facade (demo-scale negligible).
      - **Chunk 2 DONE:** the **conformance-through-wasm** gate
        (`conformance.test.mjs`) drives every `mode: run` fixture from doodle-rust
        through the wasm surface, matching transcript + raise (message + line:col)
        against the fixture directives (replicating native `matcher.rs`) — the 4 run
        fixtures pass bit-identically (§4.1). **doodle-web CI** (`ci.yml`) is the
        cross-repo gate: checks out both repos as siblings, builds the wasm
        (Cargo.lock-matched `wasm-bindgen-cli`), runs pump + conformance in Node.
        12 Node tests. (As the run-fixture corpus grows, the gate widens.)

- [x] **M3.6 — turtle canvas renderer + animated `forward` + S-23. DONE**
      (doodle-web `a95d026` + `8ec4cb1`; doodle-rust `6ab0927`; discussions
      `aac25d3`). New pkg **`@doodle-lang/turtle`**: animated `draw_line` over an
      injectable frame clock, instant `set_turtle`/`clear_canvas`, a two-layer
      surface (interrupted `forward` discards its trail), `CanvasSurface` +
      `turtleToPixel`. **S-23** stop-mid-`forward` → `Faulted(Cancelled)`. Review
      surfaced a **MAJOR** cancel-vs-raise asymmetry → **fixed in the engine**
      (`6ab0927`): a pending cancel discards a resolve-with-raise and faults
      `Cancelled` (E§10.1 pin). 24 Node tests + 2 doodle-core regression tests.

- [x] **M3.7 part 1 — Lezer grammar + §6.4 parity gate. DONE**
      (doodle-web `c9911bf`; corpus doodle-rust `96392c0`). New pkg
      **`@doodle-lang/lezer-doodle`** (IDE grammar, CSP-clean static parse table).
      External newline tokenizer + bracket context (§3.2 continuation);
      engine-mirrored S-4 do-attachment, two-word `else if` elif, separator-less
      members, docstring-only record body. **§6.4 gate:** 20 `mode: static,
      stage: parse` fixtures (`lang/LA/gp-*`) classified identically by the Lezer
      grammar (parity test, cross-repo) and the engine. 3-lens review + a
      Lezer-vs-engine divergence harness found+fixed 3 false-positives.
      - **Documented CFG omissions** (engine-only checks): lvalue validity,
        module-level placement, positional-before-keyword arg order, docstring
        placement. **Known gaps** (corpus excludes): block PARAMETERS `do (x,y)`,
        string-internal validation (interpolation/escape/margin), non-ASCII idents.
- [x] **M3.7 part 2 — CodeMirror LanguageSupport + highlighting. DONE**
      (doodle-web `9cb72d4`). `@doodle-lang/lezer-doodle` exports `doodle()` (a
      CodeMirror 6 `LanguageSupport`) + `doodleLanguage` with complete `styleTags`
      (coverage-probed) + a headless highlight test. Still CSP-clean.
- [x] **M3.7 part 3 — the demo app (editor + canvas + run/stop). DONE**
      (doodle-web `520b77c`). New app **`@doodle-lang/demo`** (Vite static site):
      CodeMirror editor w/ Doodle highlighting, turtle canvas (`CanvasSurface` +
      rAF), Run/Stop wired to the pump via a DOM-free **run core** (`src/run.ts`,
      Node-tested: draws + streams `print`, load-error returned, Stop →
      `Faulted(Cancelled)`), a print output pane, status line. Vite picks up the
      wasm as an asset (`loadEngine()` fetches it; ~113 KB gzip bundle). Strict CSP
      meta (no JS eval; `wasm-unsafe-eval` for wasm). **Playwright smoke** drives
      the built app in headless Chromium (mount/Run-draws/Stop-halts), in CI. R8
      note in a page footer.
- [x] **M3.7 part 4 — live line highlight. DONE** (doodle-web `51d7fb7`; engine
      doodle-rust `77ceeb5`). Highlights the executing **user-program** line as the
      turtle draws. Wrinkle: a user `forward(10)` runs `draw_line` *inside* the
      prepended library, so the top frame is in the prelude — added engine
      **`currentUserSpan()`** (`Instance::call_site_spans()`: innermost call site at
      or past the prelude, else the top frame if user code). Pump samples it into
      `onPosition`; demo maps span→line → `onLine` → a CodeMirror decoration
      (cleared at run end). Node-tested (`forward\nright\nforward` → [1,2,3]) + a
      browser highlight test. **M3.7's page is functionally complete.**
- [x] **M3.8 — deploy pipeline + public URL + posture gates. DONE** (doodle-web
      `b4e0653` + `73f6e9b`). **Live: https://doodle-lang.github.io/doodle-web/**
      (GitHub Pages, Actions source, HTTPS, unlisted — Decisions #3/#4 RESOLVED
      2026-08-23). `deploy.yml` builds the wasm + libs + `vite build
      --base=/doodle-web/` and publishes on push to main. Release gates in
      `ci.yml`: first-load budget (~349 KB gz / 460 KB), D-7 posture check
      (self-contained, meta CSP no JS eval, no trackers), `npm publish --dry-run`.
      Verified live headlessly (styled editor, Run draws, Stop halts, zero console
      errors). Caught in live verify: `style-src 'unsafe-inline'` needed for
      CodeMirror's injected styles (script-src stays strict); a browser test now
      guards zero console errors. Deferred (Cloudflare, #3): HTTP-header CSP +
      frame-ancestors + `doodle-lang.dev`.
- [x] **M3.9 — M3 exit review + accept-criteria walk. DONE — M3 COMPLETE.**
      (doodle-rust `3517f48`, doodle-web `2d9619a`+`1e45522`.) **Accept #1–#5 all
      pass**: live headless walk of https://doodle-lang.github.io/doodle-web/ (editor
      styled 394 ms, spiral first draw 233 ms after Run, exec line highlights, Stop →
      `stopped` in 109 ms, zero console errors, nav→running 636 ms) + CI gates green
      (wasm ship-size 179 KB / 300 KB brotli; conformance-through-wasm incl. `print`).
      **S-15-in-E**: already fully recorded at M3.3 (E§7.6/§7.2/§5.4 + App C);
      lens D re-verified spec-vs-code parity — no new edit needed. **Multi-lens review**
      (ultracode workflow, 6 read-only lenses → per-finding adversarial verify; pump/stop
      +S-23, S-15, and turtle/line-highlight came back clean): 4 confirmed findings —
      - **#3 (major) FIXED** (doodle-web `1e45522` then corrected in `9ee4ab0`): the
        ≤300 KB brotli gate measured a `wasm-opt`'d artifact while the ship path stopped
        at wasm-bindgen, so the gate measured a binary the pipeline never shipped and
        falsely documented itself as measuring the shipped bytes. **Fix:** new
        `wasm-ship-size.sh` gates the **exact raw shipped bytes** (193 KB brotli, under
        budget), wired into ci.yml + deploy.yml. **Correction:** the first attempt added
        `wasm-opt -Oz` to the ship path (the user's chosen shape, ~179 KB); it passed
        locally but the **apt binaryen on CI mis-optimized wasm-bindgen's growable
        function table** → a runtime `WebAssembly.Table.grow()` failure on every turtle
        program. The conformance-through-wasm CI job (now driving the optimized bytes)
        caught it, but **deploy does not run that suite, so it shipped the broken binary
        and briefly took the live demo down**. Reverted to raw wasm (`9ee4ab0`) to restore
        the site, then **re-enabled `wasm-opt -Oz` SAFELY** (`4e9d9f5`, user-directed):
        a **pinned binaryen 130** (`scripts/install-binaryen.sh`, sha256-verified, replaces
        the miscompiling apt one) **+ a deploy-time smoke test** (`npm test -w
        @doodle-lang/engine` drives the optimized bytes before publishing, so a bad
        optimization fails the deploy instead of shipping). Shipped wasm is now the
        optimized **179 KB brotli**; CI + Deploy green, live site verified 7/7.
      - **#2 (minor) FIXED** (doodle-rust `3517f48` + doodle-web `2d9619a`): the pump's
        int decode was non-total — a bignum capability arg (beyond i64) threw out of the
        drive and wedged the instance (reachable via `pencolor(10**30,0,0)`+`forward`).
        Added a total decimal-string int boundary (`makeIntStr`/`asIntStr`); the pump uses
        it. Tests added (Rust boundary/facade + pump e2e).
      - **#1 (nit) TRACKED** — conformance-through-wasm gate never crosses a slice
        boundary (all driven fixtures finish inside the 100 k default fuel), so it doesn't
        exercise the slice/resume path. Backstopped by doodle-rust `drive_directives.rs`
        slice-parity suite (same doodle-core compiles to wasm). Coverage gap, not a
        behavior bug. **Fix when convenient:** re-run a subset of the `run` fixtures under
        `fuelPerSlice: 1n` and assert identical output + raise line:col.
      - **#4 (nit) TRACKED** — `posture-check.sh` scans only `index.html` + `assets/*.js`
        for external refs/trackers, not bundled `assets/*.css`. Backstopped by the runtime
        meta CSP (`default-src 'self'`), which blocks any external load. **Fix when
        convenient:** scan the whole built tree (incl. `assets/*.css`) for `https?://`.
      - [x] **Follow-up (from #3's correction) — re-adopt `wasm-opt -Oz` SAFELY. DONE**
        (`4e9d9f5`): pinned binaryen 130 (`scripts/install-binaryen.sh`) in ci.yml +
        deploy.yml, and the deploy smoke test below. Shipped wasm back to 179 KB brotli.
      - [x] **Follow-up — deploy publishes without a functional smoke test. DONE**
        (`4e9d9f5`): `deploy.yml` now runs the engine suite against the optimized bytes
        (turtle draw_line through the wasm) before the publish, so a wasm that does not run
        fails the deploy instead of going live.
      - **Decision #9 (R8 guard) RESOLVED (user, 2026-08-22): accept + document,
        no interim guard.** A huge `**` builds a giant bignum and freezes the
        main thread between safe points (limits poll only at statement
        boundaries; the heap limit faults only after `num_bigint` materializes
        the whole number). Web Worker / subset-restriction / deadman-timer all
        declined (too complex or silly). **Deferred follow-up (the real fix):**
        bound bignum *magnitude* enforced **during** the op (so `**`/multiply
        fault mid-computation the instant the result would exceed the size/heap
        limit) — an arith + E§10.2-limits change, likely with M4's finer limits.
        Until then: brief freeze then heap fault; the demo documents it. No
        longer blocks M3.7 wiring.

**Milestone M2b — Drive layer** (`[M]`; working plan
**`plan/plan-m2b.md`**, written 2026-08-01, decomposed into **M2b.1 …
M2b.7**). Two scope-boundary calls **ruled by the user 2026-08-01**: (1)
M2b lands the drive-state-machine plumbing + a *basic* `Step`/`Continue`
(pause at the next statement safe point; `StepInto/Over/Out` by frame
depth); breakpoints / tail-aware stepping / raise-trap-pause /
per-subexpression / inspection panels stay **M6**. (2) **S-40** bounded-run
fuel + `Paused(SliceEnd)` is **deferred to M3** (not M2b). No starred
M2b fresh-asks remain: **S-43 RESOLVED** (2026-08-01, namespace-seed
shadowable), the **post-`Raised` E§3.3 state RESOLVED** (2026-08-02,
distinct terminal `raised` state), and **S-46 DIRECTION CONFIRMED**
(user, 2026-08-02: the MD §12 `NonLocalExit` mechanism, option 1 — with
riders recorded on the plan-m2b S-46 obligation; the E§7.2/§5.4 edit +
App C S-46 ride M2b.5 per the spec-delta process).

- [x] **M2b.1 — boundary value model** (constructors + typed readers +
      `Kind`, E§4.3/§4.4). **DONE** (doodle-rust `1020795`) — `machine/
      boundary.rs`: `make_*` (int/bool/nil/float/string/bytes/list +
      `list_append`) + readers (`kind_of`/`as_int`/`as_bool`/`as_float`/
      `is_nil`/`string_bytes`/`as_bytes`/`list_length`/`list_get`), all
      handle-mediated (host-owned GC roots). Determinism on construct:
      `make_string` UTF-8+NFC, `make_float` NaN-canonicalizes (±∞/−0.0
      inert). Accounting-aware `Heap::list_push`. 5-lens review: 0 code
      defects; folded a MAJOR test-gap (`list_push` accounting now tested
      charge+sweep-reclaim) + a MINOR doc gap (`list_get` handle ownership).
      14 unit tests.
- [x] **M2b.2 — foreign-function registry + synchronous foreign fns +
      `print` (S-43). DONE** (doodle-rust `f0d4f1a`). `machine/intrinsic.rs`:
      `Registry` (register-before-load; dup/builtin-collision → `HostError`;
      order = replay identity), `Intrinsic` descriptor, `IntrinsicCtx`, inline
      synchronous `apply`, `print` + a provisional value renderer (Stringable
      stand-in → M4/M9a). `CalObj` split `CallableTarget::{Source,Intrinsic}`
      (apply runs an intrinsic inline, never a frame); object defs → new
      `heap/objects.rs`. Namespace order `globals → BUILTINS → intrinsics`
      (user shadows); output sink + `Instance::{load_with_intrinsics,output}`.
      Runner registers `print`, matches `expect-out`; **conformance 66/0/0**
      (last SKIP gone). 6-lens review: 0 code-safety defects; folded 2 MINOR
      (block-on-block-less-intrinsic now raises for call parity; a no-`expect-out`
      run fixture now checks output is empty). Foreign defaults inline-only
      (heap-backed → S-42/M7). **Discovered + wired: string-literal evaluation**
      (a real M2a `StrLit` gap; non-interpolated only — interpolation is M4).
- [x] **M2b.3 — drive-state machine. DONE** (doodle-rust `934481b`).
      `drive::run` is a resumable loop (`Ready`/`Paused` → `Completed`/`Raised`/
      `Faulted`/`Paused`) + a phase-guarded `resolve(Resolution)` (resume path
      M2b.4). **Basic `Step*`**: `step()` reports statement-level safe-point
      crossings + depth; `should_pause` = `Step`/`StepInto` (next SP),
      `StepOver` (`depth ≤ anchor`), `StepOut` (`depth < anchor`);
      `RunToCompletion`/`Continue` never pause (M6 adds breakpoints).
      **`InstanceState::Raised`** (E§3.3 outcome↔state; terminal, distinct from
      `Faulted`); host-contract phase guards. 5-lens review: **1 MAJOR folded**
      — a root-caused `StepOut` overshoot (a frame-popping `return`/`break`
      unwind reported no return safe point → StepOut ran into sibling calls;
      the settling unwind transition now reports it + runs `limits::safe_point`
      like the fall-through path — return-via-unwind now ticks a return safe
      point, consistent with E§7.4; determinism gate green). 8 drive-directive
      tests + a StepOut regression.
- [x] **M2b.4 — suspending capabilities + `resolve`. DONE** (doodle-rust
      `6905717`). `Intrinsic` gained `ForeignBody::{Sync, Capability}`; a
      capability call parks a `PendingRequest` (id = registry index, args) and
      the drive loop returns `Suspended(CapabilityRequest{capability, host-owned
      arg handles})` — no state torn down (MD §14). `resolve(Value)` injects the
      result (Void for a `to` cap) + resumes under the in-force directive;
      `resolve(Raise)` → `Raised` at the call site (**provisional `HostRaised`
      kind + rendered message per the user ruling; value-carrying is M4** — spec
      delta below). Parked args are GC roots while `Suspended`; capability id is
      stable (S-43) for replay. `read_line` capability. `Value`/index types →
      `machine/value.rs` (length). 6-lens review: **1 MAJOR folded** — a
      stale-handle `resolve` returned `Faulted` but left the instance `Suspended`
      (resumable half-state, E§3.3 violation); now cleared + terminally `Faulted`.
      12 drive-directive tests.
**M2b.5 split for size (M2a.6-scale) into 5a (done) + 5b (next).**
- [x] **M2b.5a — reentrant drives + native `each`. DONE** (doodle-rust
      `b02b5f8`). `IntrinsicCtx` → a rich mutable step-context with
      `invoke_block` running a nested drive (block frame at a `Consumer::Native`
      boundary; complete/`continue`/raise). A limit inside the nested drive
      parks `Machine.reentry_fault` (the Raise-typed `apply` chain can't carry a
      `Fault`); `step` surfaces it. Native `each` (fixed-count over the live heap
      list). `break`/`return` crossing the boundary → `Unsupported` (S-46 → 5b).
      Wired **list-literal eval** (`Node::List`, a demo-subset gap). 5-lens
      review: **2 folded** — CRITICAL: recursion *through* a native consumer
      overflowed the host Rust stack (SIGABRT) → `MAX_REENTRY_DEPTH`=64 (MD §14
      drive-depth) faults with `StackDepth`; MAJOR: the `foreign_roots` GC-root
      fix (MD §15, a use-after-free of heap-valued `each` elements) had an
      ineffective test → a `gc_every_safe_point` knob collects inside the nested
      drive. Files split (intrinsic→dir, `lifecycle.rs`). 10 tests. *Provisional:*
      `MAX_REENTRY_DEPTH`=64 flat bound (stack-aware/configurable → M3/M7);
      `Consumer::Native` unit at 5a, **5b re-adds boundary depth** for resume.
- [x] **M2b.5b — S-46 non-local exits. DONE** (doodle-rust `bfc9e6f`; spec
      E§7.6/§5.4 + App B.1 + App C S-46 RESOLVED in the same discussions push).
      `Consumer::Native { boundary }`; a `break`/`return` crossing the native
      boundary leaves the `Unwind` **parked** (`Unwind::NativeBreak`, or a
      `return`'s punch-through), the nested drive returns
      `BlockResult::NonLocalExit`, the callback returns promptly, and
      `intrinsic::apply` resumes the parked exit at the apply site
      (`resume_native_boundary`) — a `break` completes that call, a
      `return`/outer break keeps unwinding (nested consumers compose
      innermost-out). `continue` stays a normal completion. A valued `break` to
      the procedure `each` raises `NoValueDestination` (parity with the Doodle
      `to`-consumer, open S-10). **Host-contract faults (rider 3 / S-16 family):**
      driving the block again after a `NonLocalExit`, or returning a value/raise,
      → `Faulted(Internal)`. Removed the 5a `ExceptionKind::Unsupported` stub.
      Split demo intrinsics + renderer into `intrinsic/builtins.rs` (length).
      **5-lens read-only review (find→verify): 0 confirmed defects**; folded the
      one spec-parity residual (the E§7.6 host-contract fault was documented but
      not enforced → now enforced + tested). 6 new drive tests + 2 crate-internal
      host-contract-fault tests.
- [x] **M2b.6 — foreign values + finalizers. DONE** (doodle-rust `e7203cb`;
      E§4.5 finalizer-timing wording tightened in the same discussions push).
      `Value::Foreign` allocatable: a `foreigns` slab of
      `ForeignObj { tag, ptr, finalizer }`; boundary `make_foreign` +
      `foreign_tag`/`foreign_ptr` (`machine/foreign.rs`, split from boundary.rs
      for length). Inert to Doodle (identity, no fields, arithmetic raises).
      **Finalizers, host-side, exactly once:** GC takes+queues a dead value's
      finalizer (run after the sweep, MD §15); `Drop for Instance` runs every
      still-live foreign's once at `destroy` (E§3.1); a GC-finalized value is
      never re-run at destroy. Each finalizer call is `catch_unwind`-isolated
      (must-not-unwind contract; a buggy one can't skip peers or abort `Drop`).
      Determinism: the finalizer is uncounted host state (fixed
      `foreign_payload`), can't re-enter the instance. **5-lens read-only review:
      1 confirmed folded** — a panicking finalizer leaked its peers / could abort
      in `Drop` → `catch_unwind` isolation + test. 9 foreign tests (7 lifecycle +
      isolation + a Doodle-injection inert/finalize test). *Provisional (S-42-
      lite):* boxed `FnOnce(u64)` finalizer; C-ABI `extern "C" fn(void*)` → M7.
- [x] **M2b.7 — cancellation + observation + M2b exit review. DONE**
      (doodle-rust `c181f7e`). **Cancellation** (E§10.1): a cloneable/thread-safe
      `CancelToken` sets a flag polled at each safe point; once set the drive arms
      `Unwind::Cancel`, which tears the stack down one frame per transition (MD §12
      unwind path; block/`with` cleanup runs at M4 — inert now) and faults
      `Faulted(Cancelled)` — terminal, non-resumable, not catchable. Works top-level,
      across suspend/resume, and through a native consumer's reentrant drive (the
      parked `Cancel` crosses the S-46 boundary; `resume_native_boundary` declines
      it). **Observation** (E§8.1/§8.2 minimum, new `machine/observe.rs`):
      `current_position()` (module id + byte `Span`; end-of-body → end-of-module,
      never `(0,0)`), `stack_walk()` (callable handle + call-site + tail count;
      frames gained `call_site`). Step budget already wired (M2a.9). **M2b exit
      review (5-lens + re-run determinism): exit-criteria + determinism CLEAN; 3
      minor/nit folded** — real reentrant-cancel test (test-only
      `IntrinsicCtx::request_cancel`), `poll_cancel` doesn't arm on an empty stack,
      `current_position` end-of-body fallback. *Cancel-vs-completion race RESOLVED
      (user, 2026-08-03) + E§10.1 pinned/generalized:* cancellation is about future
      work — once observed, no further program work runs; it takes effect only where
      work remains, else the run's own terminal outcome stands (`Completed`/`Raised`/
      resource `Faulted`), and cancel on a terminal instance is a no-op. **⇒ Milestone
      M2b (host/embedding layer) is COMPLETE** (M2b.1–M2b.7);
      richer E§8.2 frame surface (locals, dyn-bindings) + debugger = M4/M6.

**resolve(Raise) provisional filed (M2b.4; user-ruled 2026-08-02):** at M2b,
`resolve(Raise(handle))` surfaces `Outcome::Raised` at the capability call site
with a provisional engine exception (`ExceptionKind::HostRaised` + the
host-raised value **rendered into the message**); the host-provided value is
**not** carried as an exception value yet (`Exception` has no value slot). The
value-carrying exception that `rescue` binds arrives with **exceptions-as-values
at M4** (E§9). Resolve the E§7.5 host-raise representation in the spec no later
than M4. Tracked by `a_capability_resolved_with_a_raise_surfaces_raised`
(`tests/drive_directives.rs`).

**S-51 RESOLVED (user, 2026-07-11): `(a) = 5` is accepted** — parens are
transparent around assignment targets (`'(' lvalue ')'` added to L§5.3 +
App A; rationale + language survey in App C S-51 and L App D.1). Matches
the shipped M1.7 parens-transparent parser; no code change. **Small
follow-up for the next parser work item:** a pinning accept-side test
(`(a) = 5` assigns to `a`, no diagnostic).

## Next up

**M1 is complete** (front end: lexer, parser, resolver, diagnostics; stage
gate at **Full**; M1.1–M1.15 all landed — see the Done log and `plan-m1.md`).
CI green on both repos (the M1.14 fuzz workflow was fixed 2026-07-24:
`taiki-e/install-action` needs `with: tool: cargo-fuzz`, and the fuzz build
must pin `--target x86_64-unknown-linux-gnu` — the musl prebuilt defaults
break ASan; doodle-rust `3597302`).

Milestone **M2a — Machine Core** (`[L]`; see the new **`plan/plan-m2a.md`**).
The CESK machine + slab heap/GC v0 + handles over the demo subset. The
*design* (`plan/machine-design.md` v0.2.1) is accepted (M2a gate satisfied);
`plan-m2a.md` sequences it into work items **M2a.1 … M2a.12**, dependency-
ordered. **M2a.1 landed** (heap foundation: generic `Slab<T>`, the `Heap`
with strings/bytes/lists slabs, len-based deterministic payload accounting,
serial identity, `same_ref`; no GC yet — read-only review clean, one MAJOR
folded by dropping an untracked-growth accessor; the payload-count gap is
tracked in the spec-delta queue below). **M2a.2 landed** (machine skeleton:
`Machine`/`Frame`/`Cont`/`step()` over `ResolvedModule`; the M0 `Instance`/
`drive` walk replaced; literals + statement sequencing + module-top-level
Void return; E§7.2 top-level completion resolved to `Completed(None)`).
**M2a.3a landed** (arithmetic + numeric tower — bignum promotion/demote-on-fit,
floored `//`/`%`, S-56 finite-result rule; the raise path — `step` returns
`Result`, uncaught → `Outcome::Raised`; Void enforcement. Review-clean after
minor folds; `num-bigint` added; the S-56 overflowing-**literal** diagnostic has
an `#[ignore]`d tripwire, deferred to a front-end session per the user).
**M2a.3b landed** (comparison + equality per S-28 + strict booleans; the exact
int↔float comparison verified via a 1.2M-pair cross-check + a 6-lens adversarial
workflow, 0 findings). **M2a.4a landed** (module binding cells + `let`/`const`/
assignment; the carried module-cell-for-globals + forward-reference question is
**resolved** as the user-approved **TDZ** model — cells uninitialized at load,
filled in order, read-before-defined raises; verified by a 3-lens adversarial
workflow vs. the resolver, 0 findings). **M2a.4b landed** (`if`/`while`/`loop`
intra-frame control flow — strict-`Bool` conditions, construct-body locals via
`Frame::locals`, `if`-expression value discipline; review clean above NIT, one
forward note tracked for M2a.6 re: loop-body slot reset). **M2a.4 is complete.**
**M2a.5 landed** (calls: `to`/`fn`/anon-`fn` non-tail, positional + keyword args
+ defaults evaluated in the callee activation, `Callable` frames + `ReturnBarrier`,
one canonical `CalObj` per module `to`/`fn` interned at declaration-execution,
`is` + a provisional built-in type-value prelude; `NotCallable`/`ArgumentError`
raises. 7-lens adversarial workflow: 0 confirmed defects — the one finding, a `fn`
dynamically falling off the end returning Void silently, is the fn-tail-`to` case
correctly deferred to **M2a.7** (apply-time kind gate), tracked below with an
`#[ignore]`d tripwire. Two provisionals filed in the spec-delta queue: `to`/`fn`
TDZ, and the built-in type-value prelude). **M2a.6 landed** (blocks + three-tier
exits + the §12 unwind mechanism — the hardest item: block passing/invocation
(incl. **nested** block-param invocation, block composition), `BlockOuter` static
links, one `Unwind` record dispatching on the resolver's `ExitTarget` with
punch-through; S-9 landed, S-10 loop-half enforced statically. A 7-lens adversarial
workflow found 4 bugs, **all folded**: stale-register leak on value-less-tailed /
empty blocks and break-exited loops (fixed by clearing `reg` at statement
boundaries — resolves the carried statement-boundary-register question); nested
block-param invocation (fixed, MD §10 **consumer = the receiving call** refinement,
user-approved v0.2.4); valued `return` in a `to` (**C** — new
`valued-return-in-procedure` static error); block-param used as a value (**D** —
new `block-used-as-value` static error). Open **S-10 to-consumer half** raises
provisionally, see the queue). **M2a.7 landed** (PTC: the S-55 apply-time kind gate
+ frame reuse — a marked-tail call reuses the current callable frame iff the
callee's kind matches, so a tail loop runs in constant memory with `tail_count == N`;
mismatches run as ordinary frames (to-tail-fn discards, fn-tail-to falls off). The
**fn-falls-off** enforcement raises `FunctionFellOffEnd` at **both** return paths
(the `ReturnBarrier` and the `return`-unwind) — resolving the M2a.5 tripwire. A
6-lens adversarial workflow found 1 bug (a bare non-tail `return` bypassing the
falls-off check), **folded**. S-55 reuse tests landed. Deferred + flagged:
block-frame tail reuse and the ring observation surface, see the queue). **M2a.8 landed**
(closures — representation B: a resolver-`cell_boxed` local lives in a heap `CellObj`
shared by the closure and its creating frame. New `machine/local.rs` (`Local::{Direct,Boxed}`,
`read`/`write`/`rebind`/`build`): assignment mutates the cell (captures observe it), a `let`
mints a **fresh** cell (loop-fresh, L§5.4), frame entry cell-boxes marked slots and splices
the closure's captured cells; `make_callable` reads each `CaptureSource` from the creating
env into `CalObj.captures`. A 6-lens adversarial workflow found 1 MAJOR — **letrec**: a
self-recursive nested `fn`/`to` captures its own cell-boxed slot, so `define_callable` must
allocate the binding's fresh cell **before** `make_callable` reads it, then fill it, else the
self-call derefs a stale uninitialized cell → spurious `UsedBeforeDefined`; **folded** with a
regression test (recursive `fn` `fact(5)=120` + recursive `to` mutating a captured counter).
Exit criterion 5 met — `make_counter`/S-11, shared bindings, loop-fresh, block-capture hops>1,
nested closures all verified. **Cells become GC roots at M2a.10** (a cell-boxed body local
allocates a throwaway entry cell its first `let` replaces — harmless until GC)). **M2a.9
landed** (safe points + fused counter + limits — statement-level safe points (`Seq`/call
entry/return) evaluated in `machine/step.rs` around an extracted `dispatch()`; `step()` now
returns `Result<(), Halt>` (`Halt = Raise | Fault`), `drive::run` maps to `Raised`/`Faulted`.
New `machine/limits.rs`: `FusedCounter` (MD §9 decrement-and-branch; step budget the sole
contributor now) + `safe_point`/`check_stack_depth`. Call entry = frame-stack growth, so
tail-call reuse is exempt (L§8.7). Limits (E§10.2): step budget, heap (`bytes_allocated`),
non-tail stack depth → `Faulted(LimitExceeded(kind))`; new `drive::Limits` config subset +
`Instance::load_with_limits`. **Heap object-count hole closed (user Option A, machine-design
v0.2.5):** a fixed per-object overhead in `bytes_allocated` (`heap.rs` `charge_object`), so an
empty-object flood faults instead of OOM. 6-lens read-only review: 0 confirmed (2 refuted;
`**` single-transition overshoot documented + tracked). Exit criteria 2 + 3 met; the
statement-boundary register pin was already resolved at M2a.6. `tests/limits.rs`). **M2a.10
landed** (GC v0: precise non-moving mark-sweep — `heap/slab.rs` mark bit + index-order sweep;
`heap/gc.rs` work-stack tracer + per-slab sweep with alloc/sweep-shared payload accounting;
`machine/gc.rs` root enumeration (frame callables + locals + conts via `Cont::each_value`
[exhaustive], `reg`, `unwind`, ring callables, namespace cells); trigger in `safe_point` at
`min(gc_threshold, heap_bytes)` — GC threshold floor 1 MiB / `live×2` re-arm, **or** the heap
limit as the MD §15 last-ditch collect, then fault only if still over. 5-lens read-only review:
1 confirmed major (missing last-ditch collect → spurious `Heap` fault under a limit below the GC
floor), **folded** + regression test; roots/determinism/accounting/precision lenses clean. Cells
and ring callables are now GC roots (closes the M2a.7/M2a.8 forward notes). *Accept met:*
forced-GC-every-step never changes results, garbage reclaims to baseline, garbage loop bounded
under normal and below-floor heap limits. Handle roots → M2a.11; `dyn_stack` → M4; the full
GC-stress corpus run → M2a.12). **M2a.11 landed** (handles + instance config: `machine/handle.rs`
`HandleTable` — generational `(index,gen)` u64 handles, `intern`/`retain`/`release`/`resolve`
with the boundary generation check; live handles' values are GC roots. **S-41 resolved (user,
Option A):** `drive::Config{limits, unicode_version: Option<UnicodeVersion>}`; `Instance::create`
validates the requested version against the build-pinned `unicode::UNICODE_VERSION` (read from
`unicode-normalization` = Unicode 17.0.0), failing loud on a mismatch; `None` uses the pin. Spec
updated E§3.1 + Appendix C S-41 RESOLVED. 4-lens review: 1 folded (minor) — a compile-time
cross-check now fails the build if `unicode-ident` (identifier lexing) ever skews UCD from
`unicode-normalization`; handle/GC/determinism lenses clean. Deferred: debug cross-instance handle
id → M5, typed host readers → M2b). **M2a.12 landed — MILESTONE M2a (MACHINE CORE) COMPLETE**
(GC-stress determinism harness [broad corpus driven normal vs GC-every-safe-point, bit-identical
terminals — exit criterion 4]; conformance **run executor arm** [`matcher::run_dynamic` matches
`expect-raise` message+position; `main::needed_capability` skips run tests needing `print`];
stage gate bumped `Full → Run` atomically; three raise fixtures — suite 65/0/1. A 4-lens M2a exit
review walked all five criteria: run-arm/stage/criteria-1–3 clean, two minor harness-coverage gaps
folded [bignum content now folded to exact-compared scalars, incl. one carried through a `break`
unwind]. All five exit criteria met). **Next: M2b** (the host/embedding layer — foreign-function
registration, `print` and capabilities, the drive-state machine for resume/suspend, reentrant
drives; then `expect-out` conformance run-tests execute). Spec-delta obligations in M2a are listed
in `plan-m2a.md` (E§7.2 **[done]**, S-9 **[done]**, S-55 reuse tests **[done]**, **S-41 [done]**;
plus S-10's `to`-consumer half and S-12's huge-exponent half — **S-28, S-56, S-9, S-10's loop half,
the M2a.1 heap object-count gap, and S-41 are RESOLVED**, see the spec-delta queue).

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
  - [x] **M1.10c** — value discipline + captures + stage gate. COMPLETE — all
        pieces below landed; **M1.10 (Resolver) is DONE**.
    - [x] **fn-falls-off-end (S-5)** — the four-way tail classifier
          (produces/diverges/value-less/indeterminate), `loop`-divergence via
          `loops_with_break`, `to`-call tail = Void, `fn` bodies only, tail-`while`
          suggests `loop`. Review confirmed sound; caught `return <void>` mislabel
          (fixed — `return expr` produces; its operand's Void-ness is S-6).
          doodle-rust `d82e134`.
    - [x] **Void / S-6 consuming-site check** — DONE. A post-pass (`voidcheck.rs`)
          reports a same-module `to` call used where a value is required —
          `procedure-in-expression` (L§6.11), producer-site blame naming the proc;
          static subset = a callee resolving to a module-level `to`, directly or
          the Void propagated up through an expression-position `if`/`try` (parens
          are transparent). All the ruling's consuming sites incl. the callee and
          `return`/`raise`/`break`/`continue` operands (the ones S-5 deferred here).
          The S-5 `return p()` test flipped to `procedure-in-expression`. 31 resolver
          tests. Two read-only reviews: no critical/major. doodle-rust `bb98437`.
          **Two scope decisions RATIFIED (user, 2026-07-17):** (a) local `to` →
          runtime stands, **as a normative rule**: the L§8.4/§6.11 edit must
          state the static subset as "callee resolves to a *module-level* `to`
          of the current module" (S-5's no-more-no-less demands it be spec
          text, not an implementation shortfall; App C S-6 updated); carrying
          local kinds later = a spec change. Requirement: a test pins that
          rule 2a still catches assignment to a *local* declaration (enforced
          during the scope walk). (b) The re-chunk into if-expr value
          discipline stands — sequencing, not semantics; the no-more-no-less
          audit runs at M1.10 completion, after c.
    - [x] **`if`-expr value discipline (L§6.8/§6.9)** — DONE. `voidcheck.rs`
          extended: a value-position `if`/`try` must produce on every branch —
          a missing `else` → `if-expression-missing-else`; a present branch/body
          whose tail is value-less (a `let`/`while`/`with`/assignment/`const`
          tail, a `loop` with a bound `break`, an **empty** branch/body, or a
          nested value-less `if`/`try`) → the new `non-producing-branch`
          (reserved in the rubric; `try` bodies have no `else`, so it can't reuse
          the missing-else slug). `block_void_cause` mirrors the S-5 lattice.
          A 3-lens read-only find→verify **workflow** (11 agents) caught a MAJOR
          (empty branch/body escaped the check — `block_void_cause` `?`-returned
          on an empty block, unlike `tailcheck`'s `tail_of_block`) + a slug-
          reservation MINOR, both fixed; NITs folded. 32 resolver tests.
          **Spec clarification CONFIRMED (user, 2026-07-17; recorded in App C
          S-5):** a non-local exit **diverges with respect to any consumer
          other than its own target** — `raise`/`return` (bare or valued)/
          `break`/`continue` as a value-position branch tail are fine (the
          Kotlin/Rust guard idiom; runtime-sound: the unwinder discards the
          pending bind). Bare `return` is value-less at exactly one place —
          the fn's own tail, where the judged consumer IS the return's
          target. voidcheck implements this; pin the sentence in the
          L§8.4/§6.11 edit.
    - [x] **closure captures** (representation B) — DONE. Cross-`fn` refs resolve
          to a cell-boxed capture slot (`LocalSlot`/`BlockOuter` + `cell_boxed`);
          `CallableInfo` gains `cell_boxed`/`captures`; `deferred_captures` dropped.
          `CaptureSource { slot, from: CaptureFrom { hops, slot } }` ratified — a §7
          static link from the *creating* frame (hops=0 = its own slot; hops>0
          chases the block `defining` chain). `resolve_capture` (new `walk/refs.rs`)
          threads the cell through each intervening closure (dedup by origin;
          pass-throughs get a slot), `debug_assert`s the totality invariant (chase
          through Block frames only). §8 states the invariant. Tests: helper-in-block
          (hops 1 for the fn local, 0 for the block local) + deep
          closure-in-block-in-closure-in-block threading. doodle-rust `560e4d1`.
          **Review (3-lens find→verify workflow, 8 agents) caught a MAJOR:** a param
          default naming an enclosing-fn local was resolved against the wrong frame
          (defaults were resolved before opening the callee frame) — a default runs
          at call time in the closure's activation (L§8.2), so it must **capture**.
          Fixed: alloc param slots first (0..n), resolve defaults with the frame
          current but params not yet scope-visible, then bind param names. Plus a
          "trailing"-wording MINOR (capture slots are discovery-order, not a
          contiguous suffix; splice by explicit slot) + two NITs folded.
    - [x] **stage-gate bump Parse→Full** — DONE. `implemented_through()` →
          `Some(Stage::Full)`; `full_to_diagnostics` (lex→parse→resolve, skipping
          resolve on a partial AST) + the conformance-runner `Full` executor arm
          (atomic with the gate); 8 `stage: full` fixtures over the resolver
          diagnostics. Conformance 45/0/1 (`mode: run` still skips). Two read-only
          reviews: no critical/major. doodle-rust `32852b9`.

- [x] **M1.11 — shadowing warning + tail marking (S-45).** DONE.
  - [x] **shadowing warning (L§5.1)** — DONE. A nested declaration hiding an
        outer binding emits a `shadowing` **warning** (doesn't fail loads). Covers
        `let`/`const`, the `rescue` name, **params + block params** (caret at the
        callable/block decl node — `Param` has no span), and treats **module
        globals as whole-scope** (hides a global declared later too) via a
        pre-pass `collect_global_names`. Split `walk/decls.rs` out (length). A
        read-only review caught two completeness MAJORs in the first draft
        (params bypassed the check — the rubric's flagship case; forward-global
        missed) — both fixed. doodle-rust `30701c3`.
        **Deferred (no `Param` span):** duplicate-params and a precise param/
        block-param/rescue-name caret — the warning points at the enclosing
        decl. Would need the parser to carry per-param + rescue-name spans.
  - [x] **tail marking (L§8.7, machine-design §11 + S-45)** — DONE. Marks each
        call node *tail* iff its value is the callable's result with no pending
        work, it is not inside a `with`/`try` body, and it passes **no block
        argument** (S-45). Output is a **per-node `tail_calls: Vec<bool>`** on
        `ResolvedModule` (parallel to `resolutions`/`exit_targets` — machine-design
        §2 groups tail marks with the node annotations; O(1) at the call site for
        M2a). Beyond fall-through tails, the walk also marks a `return expr`
        operand wherever it sits (a `return` abandons surrounding work), and
        treats `with`/`try` bodies, block args, and nested callables as
        boundaries; a `return` in a loop body (same frame) IS tail, but a `return`
        crossing a block boundary is not (non-local exit). New `walk/tailmark.rs`;
        annotated-corpus tests in `tests/resolve.rs` (`tails` helper). L§8.7's
        non-tail list amended for S-45. doodle-rust `c5a64a9`.

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

- [x] **M1.12 — Golden corpus: every example in L.** DONE (doodle-rust `175de57`;
      fence tagging in discussions `30b20aa`).
  - [x] **Fence-tagging + orphan-fence fix.** Tagged all 54 code fences in
        `spec/language.md`: 16 `doodle` (15 well-formed; #44 a `doodle`-tagged
        ellipsis placeholder), 32 `grammar` (EBNF), 6 `text` (keyword/operator/
        reserved-word lists). Deleted a stray orphan fence at EOF. Classification
        ground-truthed via `parse_program`, twice adversarially reviewed — zero
        mis-classifications.
  - [x] **Extraction script + committed manifest.** `scripts/lang-corpus-sync.py`
        (in doodle-rust): one CommonMark-consistent fence scan (asserts a block
        can't be left unterminated — how the orphan fence is caught), `--write`
        regenerates, default `check` fails on any drift (block edited/added/removed/
        reordered/retagged, fixture out of sync, or hand-edited manifest metadata).
        Manifest `conformance/lang-corpus.json` records all 54 blocks (section/tag/
        sha256/disposition); #44 a documented wrap, matched by section+marker.
  - [x] **Dual fixtures for the 16 golden blocks.** 16 `spec-bNN.doodle`
        (`mode: static`, `stage: full` — all resolve clean) under
        `conformance/v0.1/lang/LSEC/`, auto-run by the conformance runner; + an
        insta AST snapshot each (`tests/lang_corpus.rs`, self-contained). #44
        (§10.3 dispatch) wrapped (`…` → `show(i)`) so single dispatch gets
        coverage. Coexists with the hand-authored fixtures (distinct `spec-b*`
        family, provenance-linked). Adversarial review (3-lens) hardened the fence
        parser + manifest check; folds landed.
  - **Decisions RESOLVED (user):** #44 wrapped; corpus in doodle-rust + a
        sync-check script; **CI wiring deferred** (a future job that pulls
        `discussions` runs the sync check).
  - **Follow-up (fixture naming brittleness):** golden fixtures are keyed by
        global block index (`spec-b<NN>`), so a mid-document spec insertion renames
        every downstream fixture (the §6.10/S-11 example insertion renamed 8). Worth
        re-keying section-local (e.g. `L<sec>/example-<n>`) so insertions stay local.

- [x] **M1.13 — Broken-syntax corpus + message review.** COMPLETE — review gate
      MET (doodle-rust `9af5f7b`).
  - [x] **Corpus + rubric-pass.** 41 → **45** hand-written broken programs rendered
        through the M1.1 renderer and snapshotted (`tests/broken_syntax.rs`); the
        rubric-pass table with the sign-off column is in
        `tests/broken-syntax/README.md`. Two read-only reviews hardened it.
  - [x] **User sign-off (2026-07-19):** all 41 rows signed `ok` (verdict + fix
        direction), plus four corpus additions pre-approved.
  - [x] **Corpus additions (rows 42–45):** the individually-ruled interpolation/
        escape diagnostics — empty-`{}` (S-48), comment-in-`{…}` (S-50),
        newline-in-`{…}` (S-47), string `\x`-one-digit (S-49).
  - [x] **Span fixes (in scope):** unclosed-construct/-delimiter errors now point
        at the OPENING token (systematic: `expect_end_span`/`expect_close` take the
        opener span; all 15 sites incl. group `(` and interpolation `{`, spot-
        checked). Duplicate-declaration gained the "the original is here" note.
        01/08/16/32/35 → PASS. Final: **30 PASS / 7 NEEDS-WORK / 8 FAIL.** A review
        caught two hand-rolled close-sites outside the first sweep → folded.
  - **Still spun off (message-quality, NOT M1.13):** pattern diagnostics +
        de-jargon; the `05` stray-`do` misdiagnosis (→ M1.9b); the `25`
        wildcard-only hedge; rubric Appendix-A code-catalog reconciliation.
        (**Cascade suppression is now DONE** — see below.)

  **Spun off from M1.13 (message-quality; NOT gating M1.13):**
  - [x] **Parser error-recovery / cascade suppression** — DONE (doodle-rust
        `587a7b4`). Panic-mode recovery: a `recovering` flag set on the first
        parser error, cleared at the next statement separator; suppresses follow-on
        *parser* errors (lexer/resolver unaffected). Same-statement cascades → 1
        message (02 5→1, 04 4→1, 11 5→1, 13 6→1, 36 2→1), cross-line residue
        reduced (19 9→4, 44 4→3); six corpus rows FAIL→NEEDS-WORK. Corpus now
        **30 PASS / 13 NEEDS-WORK / 2 FAIL**. Read-only review: sound (no primary
        lost, parser-scoped, deterministic).
  - [ ] **Pattern diagnostics + de-jargon** — recognize `=`→`==`, missing comma
        (call + list), stray `do`, misplaced `else`, keyword-as-name, extra `.`;
        replace `expected a statement separator`/`expected an expression` with
        plain kid wording + a "did you mean …?" fix.
  - [ ] **Rubric Appendix-A reconciliation** — the code catalog in
        `plan/error-message-rubric.md` drifted from shipped slugs
        (`assign-to-undeclared`→`undeclared-assignment`, `bad-escape`→
        `unknown-escape`, the three `*-outside-*`→`misplaced-exit`,
        `function-missing-value`→`function-falls-off-end`,
        `under-indented-line`→`margin-mismatch`); update the catalog to match.

- [~] **M1.14 — Parser fuzz targets + soak.** Targets DONE (doodle-rust `5e08b97`).
  - [x] **Three cargo-fuzz targets over the front end** — `lex`/`parse`/`full`
        (the last incl. the resolver), replacing the M0 `fuzz_smoke` placeholder;
        fuzz crate on its own workspace (nightly/ASan off the pinned-stable build).
        Smoke soak clean (60s each: 3.8M / 1.4M / 2.0M runs, zero crashes).
  - [ ] **1 h exit-criterion soak** — partial run clean (23k-input corpus, **zero
        crashes**; corpus preserved for a resumed run), but the contiguous 1 h run
        was stopped early by the environment's background-duration cap. Complete it
        (in chunks — libFuzzer resumes from the corpus) when closing M1.15.
  - [x] **CI fuzz-smoke job + reproducibility pin — DONE** (user-authorized).
        `.github/workflows/fuzz.yml`: a 60 s cargo-fuzz smoke of each target
        (lex/parse/full) on push/PR; nightly pinned in `fuzz/rust-toolchain.toml`
        (`nightly-2026-07-09`, read by CI), `fuzz/Cargo.lock` committed.

- [x] **M1.15 — M1 exit review. DONE — M1 effectively complete.** (Only residual:
      the contiguous 1 h fuzz soak, env-limited, run-at-release — not a code/spec gap.)
  - [x] **Exit criteria walked:** (1) every L example → golden AST ✅ (M1.12);
        (2) reviewed span-correct broken-syntax corpus ✅ (M1.13); (3) every
        static-error class has a position-asserting test ✅ (audit **22/22**);
        (4) fuzzer survives 1 h — targets + CI ✅; formal 1 h run at release.
  - [x] **Status markers corrected** (M1.3–M1.10 stale `TODO` → `DONE`); hygiene
        green throughout.
  - [x] **S-item audit** — S-1/2/4/5/6/45 resolved+tested; S-7 M1-complete
        (runtime → M6); S-3/S-27 App-C `[landed]` brackets added.
  - [x] **S-11 RESOLVED** — the §6.10 closure-capture edit landed (minimal-review
        folded: `append`, non-reassignable), extracted to the golden corpus
        (`spec-b25`); App-C marked landed. (No mutation *test* — M2 runtime;
        behavior already implemented via capture representation B + machine-design
        §7.)

**Stage gate — now at Full (M1.10):** `implemented_through()` returns
`Some(Stage::Full)`; the conformance runner executes `stage: lex`/`parse`/`full`
(matcher `run_static`, parametrized by stage — lexer / `parse_to_diagnostics` /
`full_to_diagnostics` = lex+parse+resolve). The remaining bump (`run`, M2) must
likewise co-land its executor arm in `tools/conformance-runner/src/matcher.rs`
atomically; `mode: run` tests skip until then.

**M2a gate:** satisfied — `plan/machine-design.md` v0.2 accepted by the
user 2026-07-10. (Mechanism changes still require revising that document
first.)

## Spec-delta queue (near-term)

Full backlog: `plan/implementation.md` Appendix C (S-1…S-56; the
appendix is the tracker of record until GitHub issues open). Due with M1
work items (resolve in spec *before/with* the implementing item —
pairings in `plan-m1.md`): S-1 (M1.2, done), S-2 (M1.3, done), **S-3
(M1.5 — margins resolved with the user 2026-07-10: exact-prefix; empty
lines exempt, whitespace-only nonempty lines are content lines; preserve
trailing whitespace; no line-join; no comment after opener — full text in
App C; M1.5 lands the L§3.6.4 edit)**, **S-4 (M1.7, done — no-trailing-block
header parsing; L§6.4/§7.6 + App D.1)**, **S-5 + S-6-in-full
(M1.10 — code landed doodle-rust `66fde25`; L§5.3/§6.8/§6.9/§6.11/§8.4/§8.5
+ App D.1 edits landed discussions `ef7ce71`)**, S-7 (M1.9), S-11 (M1.10),
S-27 (M1.8, user decision), S-45 (M1.11), **S-47/S-48/S-49 (M1.4:
interpolations never contain line terminators in any string form; empty
`{}` is a lex error; closed escape set + `\xHH` = U+00HH in strings —
resolutions discussed with and agreed by the user 2026-07-10)**. Later:
S-41 by M2a; **S-9 — spec LANDED 2026-07-29** (punch-through with
restoration, per the pre-approved MD §12 resolution; see below);
**S-46** (non-local exits through native consumers) by M2b.

**S-28 RESOLVED (user, 2026-07-28): total/reflexive numeric `==`
(SameValueZero-style).** `-0.0 == 0.0 == 0`; exactly one NaN, equal to
itself — the engine canonicalizes NaN bits (`0x7FF8…`) at `make_float`
and at every NaN-producing op (cross-host replay requirement, E§11); NaN
ordering raises; dicts keep the first-inserted of `==`-equal keys; hash
coherence pinned (`-0.0` hashes as integer `0` via the integral-float
path; NaN one fixed constant). Spec landed with the ruling:
L§4.13/§4.8/§6.6/§15-hook-2 + App D.1; E§4.3 (Floats normative) + §11 +
App B.1; machine-design §3/§19 (v0.2.2); App C S-28 (full rationale).
Code follow-ups: **M2a.3** (comparisons), **M4** (dict-key hash).
**S-56 filed** (float overflow / nonfinite arithmetic results — raise vs
IEEE-propagate; raise is the stated lean), due **M2a.3** with S-12 —
**resolved 2026-07-28, next entry**.

**S-56 RESOLVED (user, 2026-07-28): float arithmetic raises on any
nonfinite IEEE result** — overflow to ±∞ and NaN-yielding indeterminate
forms; underflow to subnormal/±0.0 stays fine. Result-based: finite-
from-nonfinite allowed (`1.0 / ∞` is `0.0`); int→float widening
included (a bignum beyond Float range in mixed arithmetic raises;
comparisons unaffected — exact, S-28); `%` with finite operands can't
trigger; the overflowing float *literal* (`1e999`) is a **static
error**. Invariant: every machine-produced Float is finite;
host-injected nonfinites (E§4.3) are inert data — storable, comparable
per S-28, valid dict keys — with arithmetic on them raising iff the
result would be nonfinite. Closes S-12's `0 ** negative` corner by
subsumption (`0.0 ** -1` → IEEE ∞ → raises); S-12's huge-exponent
resource half stays open. Spec landed: L§4.2 + §3.6.2 + App D.1; E§4.3
note; machine-design §3 finite-float invariant (v0.2.3); App C S-56 +
S-12 update. Code: M2a.3 arithmetic finiteness check (+ conformance
fixtures); the literal diagnostic in the front end (next front-end
session).

**S-10 loop half RESOLVED (user, 2026-07-29) + S-9 LANDED: valued exits
and `with`/`try` punch-through (one L§7.10 edit).** A valued
`break`/`continue` targeting a `while`/`loop` is a **static error**
(loops yield Void — no destination), condition-blind, misplaced-exit
family, with an intent-offering diagnostic (assign before exiting /
`return expr` / use a block-consuming call); `return expr` in a `to` is
now likewise *stated* as a static error (the spec had pinned the rule;
no battery check enforced it). S-9 landed per the pre-approved MD §12
resolution (plan-m2a's ratification note): a `with`/`try` body is not
an exit destination — exits resolve through them to their lexical
target, each crossed `with`'s binding restored during the unwind,
`rescue` never catching a non-local exit; corollary: `break`/`continue`
in a `with` body with no enclosing loop/block is misplaced-exit (was:
"exits the body" — matches the shipped resolver, which pushes no Ctrl
tier for `with`). The `with` case of the S-10 ruling discussion is
thereby mooted (the rule sees through to the loop or block). Spec
landed: L§7.10 rewrite + §7.8/§7.9 cross-notes + App D.1 ×2; App C S-9
RESOLVED + S-10 PART-RESOLVED; MD §20.4; plan-m2a. **Still open:
S-10's `to`-consumer half** (a valued `break` reaching a consuming call
whose callee is a `to` — S-6-style split expected; fresh ask at M2a.6).
**Code follow-up (next front-end session):** the
valued-exit×destination-kind resolver check (valued
`break`/`continue`→ThisLoop; valued `return` in a `to`) + battery tests
+ conformance fixtures + a pinning test that break-in-`with`-in-loop
targets the loop; reserve the diagnostic slug. M2a.6 may `debug_assert`
the loop case (no runtime path needed).

**S-43 RESOLVED (user, 2026-08-01): namespace-seed, shadowable — the
provisional pre-module intrinsic mechanism.** Host-registered intrinsic
foreign functions (`print`, …) seed as **read-only** global cells
appended after module globals: a user's own declaration shadows an
intrinsic (matching S-39 import semantics and the M5 star-import
end-state; assignment is already a static error via the
visible-mutable-`let` rule). Registration completes **before the first
`load`**; late/duplicate registration (vs prior registrations or the
type-value BUILTINS) is a loud host-API error; **registration order is
replay-identity input**. Namespace order: globals → BUILTINS →
intrinsics. The M2a.5 type-value provisional folds under this
mechanism. Retirement criterion: the M5 prelude star-import replaces
the seeding with **no program-observable change** (the `expect-out`
fixtures must pass unchanged across the switch). Parked for M5: should
declaring over a prelude name trigger the L§5.1 shadowing warning?
Spec landed: E§5.5 provisional block; L§11.4 note; App C S-43;
plan-m2b obligation. Code: **M2b.2** implements (with S-42-lite +
S-19, as planned).

**S-29 RESOLVED (user, 2026-08-24; plan-m4 D-M4-1): record keys —
transitively-immutable content only.** Hashable iff a **value** record
with all fields (recursively) hashable; ref records and value records
sharing a list/dict/ref field are not — raise at insertion/lookup naming
the field. Principle: no shared mutable value reachable from a stored key
(the key itself included) — forced by structural `==` for both kinds +
§15 coherence. Explicit `implement Hashable` on a ref record permitted
(opt-in; implementer owns stability). Gained: an assertable dict-index
invariant; default hashing never enters a ref graph. Stdlib companion:
identity-keyed dict on serial identity. Spec landed: L§4.8 + §9.5 + §15
hook 2 + App D.1; App C S-29; plan-m4 D-M4-1/obligation. Code: M4.4.

**S-58 / D-M4-2 RESOLVED (user, 2026-08-24): engine-raised errors are one
built-in `Error` value record `(kind, message, details)`.** Stable
kebab-case `kind` slugs (1:1 with `ExceptionKind`; API — catalog in App C
S-58 + the rubric's App A), rubric-governed `message`, kind-specific
`details` dict; fields fixed now (no record defaults). Engine-level type,
prelude-level name (S-43-seeded, shadowable). Forgeable by design
(stdlib-in-Doodle raises identical values; provenance = the trace).
`make_error` for host parity. Spec landed: L§12.1 + §4.12 + App D; E§9 +
§4.3 + App B.1; App C S-58; plan-m4 D-M4-2. Code: **M4.5b**.

**S-59 RESOLVED (user, 2026-08-25): string `*` is symmetric** — `Int`
count on either side; `Int` only (a Float count raises), `0` → `""`,
negative raises **`negative-count`** (new S-58 slug, user-approved),
huge → the R8 cap; NFC result, grapheme length
non-additive. Spec landed: L§4.4 + App D.1; App C S-59. Code: **M4.7**
fixtures (both orders, the raises, zero, the flag case).

**S-58 catalog +`index-out-of-range` (user-approved 2026-08-25, M4.8a):**
list/string/bytes index outside `0 <= k < length`, negative included —
one slug (both directions are "no such position"), `details: {index,
length}`, message branches on sign with the no-negative-positions hint.
Access-miss triad by container kind: `key-not-found` (dict),
`index-out-of-range` (list/string/bytes), `no-such-field` (record).
Recorded in App C S-58. Code: **M4.8a** (`ExceptionKind` variant +
fixtures: too-large, negative-with-hint, each of the three containers).

**S-58 catalog +`invalid-utf8` (user-approved 2026-08-25, M4.8b):**
`decode(bytes)` on malformed UTF-8 — a data error (not `argument-error`),
`details: {position, byte}`; the host `make_string` keeps S-30's
error-return but carries the same position; `encode` cannot fail;
`decode` only raises (lossy decode = later stdlib fn). Recorded in App C
S-58. Code: **M4.8b** (`ExceptionKind` variant + fixtures: an invalid
sequence with position, a truncated sequence, the NFC round-trip).

**S-37 `Procedure` half RESOLVED (user, 2026-08-25): the split.** `Callable`
(umbrella, the L§4.1 category name; parallels `Number`) over `Procedure`
(`to`) / `Function` (`fn`, anonymous included); exactly one concrete test
per callable; foreign functions by the S-42-lite descriptor (`print is
Procedure`); type/protocol values not `Callable`. Grounds: one `Procedure`
called a function a procedure — L§8's own vocabulary contradicted; the
machine already knows the kind; up-front checks in higher-order stdlib
code. Spec landed: L§4.12 + App D spellings + App D.1; App C S-37;
plan-m4 D-M4-5. Code: **M4.9** (`is` on `CalObj` kind + descriptor flag;
BUILTINS +`Callable`/`Function`; M2a.5 tests flip). Spellings half +
the Stringable-dispatcher pin: still M4.9.

**S-60 / D-M5-2 RESOLVED (user, 2026-08-27): import is a suspension —
the resolver is async-capable from v0.1.** `import` suspends with
`ImportRequest(path, importer)`; host resolves `Source` / `NotFound` (→
`module-not-found`, new S-58 slug) / `Raise(h)`; a bundling host resolves
immediately in its drive loop (trivial case, not a mode). Grounds: the
sync hook was E's one host interaction blocking on host I/O; async
subsumes sync at zero host burden; sync would freeze the wrong shape into
the M7 C ABI; replay gains every module's source/identity in load order;
S-15 never applies. Spec landed: E§6 rewrite + §7.5 + §11 + App B.1; App
C S-60 + catalog; plan-m5 D-M5-2 + M5.1. Code: **M5.1**.

**S-61 / D-M5-4 RESOLVED (user, 2026-08-27): `extends` chains resolve
linearly, nearest default wins.** Transitive requirements (`is Child` ⇒
`is Parent`); re-declaration down the chain strengthens or overrides
(signature must conform — S-31); explicit impl beats default, then the
nearest declaring protocol's default; a cycle is unwritable (forward
`extends` → `used-before-defined` at load; corrected 2026-08-27); one
`implement` block covers the chain (§10.2 already said so), missing
members named with their requiring protocol; chains never trigger
§10.3's ambiguity (shared ancestor counts once). Spec landed: L§10.1 +
§10.2 + §10.3 + App D.1; App C S-61; plan-m5 D-M5-4 + M5.5. Code:
**M5.5** (chain chase, conformance check, cycle/missing diagnostics,
fixtures).

**S-31 RESOLVED (user, 2026-08-27): protocol calls bind first, then
dispatch.** Bind per §8.3 against the member's declared signature
(protocol's names/defaults/block param), dispatch on the value bound to
the first declared parameter — positional or keyword; unbound first param
→ `argument-error`; a member's first parameter may not default (static);
implementations conform in arity + block param, names free, **no
implementation defaults** (static error); qualified form binds
identically. Spec landed: L§10.1 + §10.2 + §10.3 + App D.1; App C S-31;
plan-m5 M5.5. Code: **M5.5** (protocol-level binding, the static checks,
fixtures).

**S-44 / D-M5-3 RESOLVED (user, 2026-08-27): native modules carry no
`parameter` cells, protocols, or implementations.** A native module exports
foreign functions/consts/foreign values/records only; language constructs
live in Doodle wrapper modules (the turtle model); dynamic state reaches a
native function as an argument. Grounds: one implementation of `with`; the
boundary carries values, not binding machinery; S-39 always targets a
Doodle cell. Spec landed: E§5.5 + App B.1; App C S-44; plan-m5 D-M5-3 +
M5.4. Code: **M5.4** (member kinds enforced at registration), **M5.9**
(turtle wrapper).

**S-13 refined (user, 2026-08-28): the prelude is an ordinary wildcard.**
Precedence at a bare use: own declarations → selective imports → all
wildcards, prelude included (no special tier); a prelude/wildcard
distinct-binding collision is `ambiguous-import` at use (names "prelude";
fix: selective import or qualified call); two wildcards aliasing the same
exporter cell are one binding (required for prelude re-exports). Rejected:
prelude-wins, wildcard-wins. Spec landed: L§11.2 (the S-13 rule itself,
previously App-C-only) + §11.4 + App D.1; App C S-13; plan-m5 M5.8. Code:
**M5.8** (+ cell-identity dedup if needed).

**D-M5-6 RESOLVED (user, 2026-08-28): hiding a prelude name warns.** L§5.1
now counts wildcard-supplied names (prelude included) as outer bindings for
the shadowing warning. **Follow-up (post-M5.8, due before M6):** a
post-resolve load-time pass diffing `ResolvedModule.globals` (+
`slot_names`) against the prelude's exports → `shadowing` warning at the
declaration span; no resolver-API change (the prelude is registered before
first `load`). User-wildcard shadowing: import-time or linter, later. Spec
landed: L§5.1; App C S-43 (parked question closed); plan-m5 D-M5-6 + M5.8.

**S-62 / D-M6-1 RESOLVED (user, 2026-08-29): fine observation mode =
non-leaf subexpression completions, observation-only.** Defined by
syntactic form in E§7.4 (operator applications, calls besides
entry/return, field/index steps, if-expr branch results, interpolation
pieces; leaves are not safe points), realized at existing continuation
boundaries; the host sees the completed span + the value just produced
(§8.4). GC/limits/cancellation/budget/fuel stay at statement safe points
in every mode (S-20 extended) — same fault instant either way; the set is
replay identity. Spec landed: E§7.4 + §8.8 + App B.1; App C S-62 + S-20.
plan-m6 D-M6-1: mark resolved in the M6 session's (untracked) plan file.
Code: **M6.6**.

**S-63 RESOLVED (user, 2026-08-29): one instance load-diagnostics record.**
Monotonic, instance-scoped; every front-end diagnostic for every module
loaded or attempted (entry at `load`, imports mid-drive), errors included
(errors keep `LoadError`/`module-load-error` as control flow); pull-read
`load_diagnostics(since?)` after `load` or at any stopped state; schema +
deterministic ordering pinned; replay-stable; not program data. Closes the
three M1.1 discovered deltas + D-M5-6's channel. Spec landed: E§3.2 + §8 +
App B.1; App C S-63. Code: **M6** — plan-m6 should fold the accessor +
imported-module accumulation + the D-M5-6 pass into the item that builds
the IDE's diagnostics surface (the M6 session's call which).

**S-58 catalog (user, 2026-08-30): `argument-error` → four kinds; details
schema pinned.** `missing-argument` / `unknown-keyword` /
`duplicate-argument` / `too-many-arguments` (one fact per slug; `details`
carries data, never a sub-taxonomy — the catalog stays one level).
Rich details ruled: `type-mismatch` `{operator, expected, got}`,
`undefined-ordering` `{operator, left, right}` (+ `nan`), `not-callable`
`{type}`, `unhashable-key` `{type, field?}`; type names are strings
spelled as the type values. Spec: L§10.3 + S-31 mentions →
`missing-argument`; App C S-58. Code: **M6.0b** (helpers pick a kind;
S-42-lite + S-31 binding share the four; rubric App A column).

**S-21 RESOLVED (user, 2026-08-30): breakpoints address `(canonical_id,
line)`; pending + re-resolve-on-load.** Canonical id is the boundary
identity (ModuleIds internal); unloaded/never-imported targets are
pending, not errors (set-then-run must work; reactive setting is racy);
every load of a canonical re-resolves (snap-forward, first-on-line) —
also the M9b reload rule; `breakpoints()` lists resolved/pending for
gutter graying; entry module's canonical id = `load`'s `module_path`
(E§3.2, owed to S-63 anyway). Spec landed: E§8.6 + §3.2 + App B.1; App C
S-21. Code: **M6.4** (pending table, re-resolution, listing, fixtures).

Discovered at M2a.3a (raise path; small, non-blocking — flag for the user):
**instance state after an uncaught raise.** E§3.3 lists ready/running/
suspended/paused/completed/faulted, with no distinct "raised" state, yet E§9
holds that a Doodle exception (`Raised`) is *distinct* from an engine
`Faulted`. The M2a.3a drive provisionally sets the instance to **`Faulted`**
after an uncaught `Raised` (the outcome carries the real result; the state is
secondary). Pin the intended post-`Raised` state in E§3.3 (add a `raised`
state, or state that `Raised` leaves the instance `completed`/terminal, or
confirm `faulted`) — batch with the M2b drive-layer work where outcomes and
lifecycle are fully wired.
**RESOLVED (user, 2026-08-02): a distinct terminal `raised` state.**
E§3.3 now lists it and pins the outcome↔state correspondence (each
stopping outcome → the same-named state, so `state()` alone preserves
E§9's raise-vs-fault line), the post-mortem surface (exception + trace
retained and observable; the stack has unwound — §8.7 trap-on-raise
observes pre-unwind), outermost-drive-only scoping (a nested drive's
`Raised` re-raises, no state change), and that cancellation stays
`Faulted(Cancelled)`. Mirrors the module-level `Failed` pattern.
Landed: E§3.3 + §9 + App B.1; plan-m2b obligation. **M2b.3 implements**
(`InstanceState::Raised` replaces the provisional; tests
raised-after-raise / faulted-after-limit).

M2a.5 provisional (flag for the user; resolve in the spec **by M2a**):
**`to`/`fn` declarations follow the same execution-order temporal dead zone
as `let`/`const`.** A module-level `to`/`fn` binds its cell when its
declaration *statement* runs (not at load), so calling a top-level callable
**before** its declaration executes raises `UsedBeforeDefined` — there is no
hoisting. This is the direct consequence of the user-approved M2a.4a TDZ
model (module cells fill in execution order) applied to callable bindings,
and is consistent with L§11.3 ("top-level code runs, in order, defining its
members") and Doodle's no-magic stance; Python behaves the same. Mutual
recursion is unaffected (bodies run at call time, after both declarations
have executed). One canonical `CalObj` per declaration is still interned
(the declaration runs once — MD §8 identity holds). **Alternative if
rejected:** hoist `to`/`fn` cells at load (fill in `Instance::load`). Pin the
rule explicitly in L (§5/§8.1/§11.3) — a one-line statement that a callable
binding, like any binding, is unavailable until its declaration executes.

M2a.5 provisional (implementation stand-in; resolve when the stdlib prelude
lands, L§11.4/§15 / E§13): **built-in type-value prelude injected at load.**
L§11.4 builds no names into the language — type-value names (`Int`, `Float`,
`Number`, `Bool`, `String`, `Bytes`, `Nil`, `List`, `Dict`, `Procedure`) are
ordinary prelude names supplied by the standard library, which does not exist
yet. So `is` could not be exercised without them; `Instance::load` seeds a
small built-in type-value prelude (`machine/types.rs` `BUILTINS`) as cells in
the namespace (appended after module globals, so a user global of the same
name wins the linear scan). Not a semantic change (the spec says these type
values exist and `is` uses them); remove the seeding when the real prelude
mechanism ships. The provisional spellings match L Appendix D (`Number`,
`Procedure` included; `Procedure` matches any callable — the proc/fn split is
still open in App D). **Folded under S-43 (resolved 2026-08-01):** the
type-value seeding and the S-43 intrinsic seeding are one mechanism (seed
after globals, read-only, shadowable; namespace order globals → BUILTINS →
intrinsics) with one M5 retirement — see App C S-43 and the queue entry
below.

M2a.5-discovered, **due M2a.7** (surfaced by the adversarial verification
workflow; already in the plan as the S-55 follow-up): **a `fn` that
dynamically falls off the end returning Void is not raised.** L§8.4/§8.7
require a `fn` reaching its completion without a value to raise **at its own
completion**, regardless of whether the caller uses the value. Today
`return_from_callable` leaves a `fn`'s register unchanged, so the fn-tail-`to`
case (a tail call to a runtime-indeterminate callee that turns out to be a
`to`) completes Void silently when the result is unconsumed; when the result
*is* consumed, the `take_value` backstop already raises. Every clean-loading
instance is a tail-position call, and enforcement wants the apply-time kind
knowledge, so it lands with the **M2a.7** kind gate (plan-m2a.md §"Spec-delta
obligations", S-55 follow-up). Tracked by the `#[ignore]`d tripwire
`a_function_that_falls_off_the_end_raises` in `tests/drive_smoke.rs`.

**S-10 to-consumer half — was OPEN (surfaced at M2a.6); RESOLVED below.** A
**valued** `break` that exits a block-consuming call whose callee is a
**procedure** (`to`) has no value destination — a `to` yields Void. The loop
half (a valued exit targeting a `while`/`loop`) was resolved as a **static
error** (user, 2026-07-29). The to-consumer half was left open ("S-6-style
split expected" — static where the consumer's kind is lexically known, runtime
otherwise). **M2a.6 provisional:** the machine raises `NoValueDestination`
(rather than silently discard the value — the loud-failure stance the loop-half
ruling took). A valued `break` into an **fn** consumer is well-defined (the
value becomes the call's result). Resolve in the spec: is a valued `break` into
a *known* `to` consumer a static error (like the loop half / valued-return-in-a-
`to`), with a runtime `NoValueDestination` backstop for a dynamically-typed
consumer? Tracked by `a_valued_break_into_a_procedure_consumer_raises_provisionally`
in `tests/drive_smoke.rs`.
**RESOLVED (user, 2026-08-25): the S-6 split — S-10 fully closed.** Static
(valued-exit family, producer-site blame, condition-blind) where the
consuming callee is lexically a module-level `to` of the current module —
S-6's boundary reused; `no-value-destination` at run time otherwise,
**attributed to the `break` site**; native consumers by S-46 parity ("a
foreign function that yields no value is a `to` for this purpose").
Unaffected: bare `break` into a `to`, valued `break` into an `fn`, valued
`continue`. Spec landed: L§7.10 + App D.1; App C S-10; plan-m4. Code:
**M4.6** (runtime half; the drive_smoke tripwire → a real fixture + the
S-46 native fixture); the static check joins the queued resolver
extension (`Ctrl::Block` carries the consuming call's static
classification).

**RESOLVED (M2a.6): statement-boundary register semantics** (was carried from
M2a.2/M2a.9). The machine now clears the result register to Void at each `Seq`
statement boundary (before running each statement; the final past-the-last step
does not clear, preserving a `fn` body's / block's value). This makes a body's
value the value of its *last* statement — Void when that statement is value-less
or the body is empty. M2a.6 forced this: a block's yield is observable, so a
value-less-tailed or empty block would otherwise leak the prior statement's
transient value; a `break`-exited loop likewise. No safe-point observer concern
remains for the register between statements. (E§8 observation of the register at
a safe point now reads Void between statements — pin in E§8 if/when the
observation surface is spec'd, M2a.9.)

**Forward note (M4), from M2a.6:** the unwinder (`machine/unwind.rs`
`block_break`/`do_return`) currently pops punched-through frames **wholesale**,
discarding their continuations — correct at M2a.6 because no `WithRestore`/
`TryHandler` conts exist yet. When `with`/`try` land (M4), the unwinder must pop
those frames **continuation by continuation**, executing each `WithRestore`
(restore the dynamic binding) and, for a raise, `TryHandler` (§12). The
loop/block-continue paths already pop conts individually.

**Deferred at M2a.7 (block-frame tail reuse; flag before it matters):** MD §11
says a **Block** frame reuses for either callee kind (a block-body tail call, or
a callable tail-invoking its own block parameter). M2a.7 implements only
**callable→callable** reuse (`reuses_current_frame` returns false for a Block
frame); a block-body tail call and a tail-invoked block parameter (`block_apply`)
fall back to **ordinary** frames — **correct**, just not constant-memory. Consequence:
a genuinely tail-recursive *block* pattern grows the stack (while-based iteration is
unaffected — the loop is constant-memory). The Callable↔Block frame conversion on
reuse (consumer becomes `TailReused`) is the subtle part. Implement as an M2a.7
follow-up or when a block-tail-recursion pattern needs it; the accept criteria
(callable self-recursion) don't require it.

**Deferred at M2a.7 (ring observation fidelity) — GC-root half DONE at M2a.10:**
the tail-elided ring (`machine/ring.rs`) is **populated** on reuse; its `callable`
refs are now **GC roots** (`RingBuffer::callables`, read by `machine/gc.rs`, §15).
**Still deferred to M2b+:** the E§8.3 observation surface reading the ring, full
S-34 fidelity (the elided frame's `call_site` span; the bounded-history semantics
tests). `consuming_serial` keeps `#[allow(dead_code)]` until that reader lands.

**RESOLVED (M2a.9; user, 2026-07-31 — Option A): `bytes_allocated` now
charges a fixed per-object overhead**, so object *count* contributes.
(Discovered at M2a.1: MD §4 charged payload only — a string's byte length /
a list's element-width — so an *empty*/tiny object charged ~0. An unbounded
loop allocating small/empty objects grew real memory while `bytes_allocated`
stayed flat, so the GC trigger never fired and `Faulted(LimitExceeded(heap))`
never tripped — an OOM-instead-of-clean-fault hole that broke exit criterion
3's "deterministic step" for that class.) The user chose a **fixed per-object
byte charge** (over a separate object-count limit): `bytes_allocated =
Σ (payload + OVERHEAD)`, a flat cross-target constant (`heap.rs`
`OBJECT_OVERHEAD = 32`, routed through `charge_object`). MD §4/§15 revised
(machine-design v0.2.5); the heap `charge_object` centralization means no
future `alloc_*` can forget the charge. Regression test
`a_flood_of_empty_objects_still_hits_the_heap_limit` (`tests/limits.rs`,
`loop do b"" end`). The related in-place-growth accounting hole — a growing
list not re-charging — is already closed structurally: the heap exposes no
raw mutable payload accessor, so growth must route through accounting-aware
methods when they land.

**Minor, tracked (M2a.9 review, 2026-07-31): single-transition heap overshoot
/ tight `**` bound.** The heap limit is checked at **safe points** (MD §15/§9),
so a single transition may overshoot the limit by whatever it allocates before
the next safe point — the limit is *soft* at safe-point granularity (an accepted
property of the model, now documented in MD §15). The notable case is
`Int ** Int`: one `arith::power` transition builds a bignum sized by the
**exponent value** (`crates/doodle-core/src/machine/arith.rs`), bounded only by
the `ExponentTooLarge` `u32`-exponent guard (~512 MB worst case), so a tight heap
limit still admits a large one-shot spike before the fault. A refinement —
estimate the result bit-size (`exp * base.bits()`) and fault/raise **before**
allocating — would bound `**` to the configured limit; it needs the heap limit
threaded into `arith` (currently only `machine/limits.rs` holds it). Not a
blocker (deterministic; bounded); implement if a tight-sandbox use case needs it.

**RESOLVED (M2a.2): top-level `Completed` value is Void.** (Was: discovered
at M0.3 — E§7.2 pinned the `Completed(value?)` payload only for a returning
`fn`, leaving a *top-level* module drive unspecified; the M0 skeleton
provisionally returned the last expression's value for observation.) The
M2a.2 machine skeleton replaced that placeholder; a top-level module drive
now completes **`Completed(None)`** — a module runs for effect (L§6.11).
Landed: `drive::run` returns `Completed(None)`; E§7.2 prose + Appendix B.1
decision entry (`spec/engine.md`); `drive_smoke` integration tests updated.

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
  **All three RESOLVED (user, 2026-08-29; App C S-63):** E§3.2's
  instance load-diagnostics record pins the warnings channel, the
  structured schema, and the ordering; read via `load_diagnostics(since?)`
  after `load` or at any stopped state. Code: M6 (the accessor +
  imported-module accumulation + the D-M5-6 pass feeding it).
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

**Cancel-vs-completion race (E§10.1) — RESOLVED (user, 2026-08-03): the
shipped choice is pinned + generalized.** Cancellation is a request about
*future* work, not a verdict on *past* work: once a cancel request is
observed, no further program work runs, and the outcome then reports what
actually happened. So cancellation takes effect only at a safe point where
program work remains; otherwise the run's own terminal outcome stands — a
cancel first observed at completion → `Completed`, first observed as an
uncaught raise reaches the boundary → `Raised`, and on an already-terminal
instance → an idempotent no-op (matching `Future.cancel()`/tokio `abort()`
conventions, and keeping the E§3.3 completed/faulted line sharp). The
observation instant is host timing, outside replay identity; a host must
accept either `Faulted(Cancelled)` or the run's own terminal outcome. The
implementation already satisfies the generalized rule (a raise/limit
short-circuits `step`/`safe_point` before the cancel poll; a terminal
instance is never re-polled). E§10.1 edit landed.

## Done

- 2026-08-31 — **S-22 riders: aux fuel is per-call host-given; non-String
  `to_string` raises as interpolation does.** Two shipped provisionals
  adjusted (E§8.4 sentence; App C S-22 bracket). Code: M6 — thread the
  fuel param (2^20 → binding default); pass-through → the `type-mismatch`
  raise.
- 2026-08-31 — **S-22 resolved + spec landed: auxiliary eval is effectful —
  only the debug context restores.** Effects persist (no purity possible);
  aux drives are replay inputs; program rails untouched; own bound,
  breakpoints/raise-trap suppressed; auto-display via §4.4 only. One
  discussions commit: E§8.4; App C S-22; this file. Matches shipped (A).
- 2026-08-31 — **E§8.8: observation mode has one axis (granularity); the
  eager-capture knob removed.** Pull inspection is inherently lazy;
  popped-frame/history state belongs to the E§11 replay track. One
  discussions commit: E§8.8; App C S-62 note; this file. M6.6:
  `ObservationMode = Statement | Subexpression` only.
- 2026-08-31 — **E§8.7 sharpened: raise-trap is directive-gated
  (`RunToCompletion` ignores it, like breakpoints).** E§7.3's stop list
  already implied it; §8.7 now says it. One discussions commit: E§8.7;
  App C S-18 bracket; this file. Matches the shipped M6 gate.
- 2026-08-30 — **S-21 resolved + spec landed: breakpoints address
  `(canonical_id, line)`, pending + re-resolve-on-load.** Ruling in the
  queue entry above. One discussions commit: E§8.6 (canonical addressing,
  pending, the listing, re-resolution, replay note) + §3.2 (entry module's
  canonical id) + App B.1; App C S-21; this file. Code: M6.4.
- 2026-08-30 — **S-58 catalog: `argument-error` split into four kinds;
  details schema pinned.** One discussions commit: L§10.3 (`missing-argument`),
  App C S-58 (catalog + schema column) + S-31 mentions; this file. Code:
  M6.0b.
- 2026-08-29 — **S-63 resolved + spec landed: one instance load-diagnostics
  record (warnings channel + schema + ordering).** Ruling in the queue entry
  above; closes the three M1.1 discovered deltas and D-M5-6's channel. One
  discussions commit: E§3.2 + §8 intro + App B.1; App C S-63; this file.
  Code: M6.
- 2026-08-29 — **S-62 / D-M6-1 resolved + spec landed: fine observation mode
  = non-leaf subexpression completions, observation-only.** Ruling in the
  queue entry above. One discussions commit: E§7.4 (statement safe points =
  accounting; the fine set by syntactic form; value-at-safe-point), §8.8,
  App B.1; App C S-62 + S-20 bracket; this file. Code: M6.6.
- 2026-08-28 — **D-M5-6 resolved: hiding a prelude name warns (L§5.1); the
  warning itself is a tracked post-M5.8 follow-up, due before M6.** One
  discussions commit: L§5.1; App C S-43; plan-m5 D-M5-6 + M5.8; this file.
- 2026-08-28 — **Name precedence pinned: the prelude is an ordinary wildcard
  (S-13 refined).** Ruling in the queue entry above. One discussions commit:
  L§11.2 (precedence + the collision rule + same-binding dedup), §11.4, App
  D.1; App C S-13; plan-m5 M5.8; this file. Code: M5.8.
- 2026-08-27 — **L§11.1 module-block form pinned per D-M5-5; S-14 closed.**
  Spec-author follow-up to M5.3: L§11.1 (file-wrapping block only — body =
  top level, docstring = module's, `Name` documentation-only; `nested-module`
  elsewhere; `exports` union + `undeclared-export`), App D.1; App C S-14;
  plan-m5 D-M5-5; this file.
- 2026-08-27 — **S-44 / D-M5-3 resolved + spec landed: native modules carry
  no `parameter` cells, protocols, or implementations.** Ruling in the queue
  entry above. One discussions commit: E§5.5 (exhaustive member kinds + the
  dynamic-state-as-argument idiom), App B.1; App C S-44; plan-m5 D-M5-3 +
  M5.4; this file. Code: M5.4, M5.9.
- 2026-08-27 — **S-31 resolved + spec landed: protocol calls bind first, then
  dispatch on the first declared parameter's value.** Ruling in the queue
  entry above. One discussions commit: L§10.1 (no-default first parameter;
  defaults live on the member), §10.2 (implementation conformance: arity +
  block parameter, names free, no defaults), §10.3 (bind-then-dispatch;
  keyword names are the protocol's; qualified form binds identically), App
  D.1; App C S-31; plan-m5 M5.5; this file. Code: M5.5.
- 2026-08-27 — **S-61 / D-M5-4 resolved + spec landed: `extends` chains
  resolve linearly, nearest default wins.** Ruling in the queue entry above.
  One discussions commit: L§10.1 (the chain rules — transitive requirements,
  re-declaration, nearest default, cycle-unwritable), §10.2 (one block covers the chain;
  diagnostic naming), §10.3 (chains never ambiguous), App D.1; App C S-61;
  plan-m5 D-M5-4 + M5.5; this file. Code: M5.5.
- 2026-08-27 — **S-60 / D-M5-2 resolved + spec landed: import is a
  suspension (async-capable resolver from v0.1).** Ruling in the queue entry
  above. One discussions commit: E§6 (rewritten around `ImportRequest` +
  `Source`/`NotFound`/`Raise`; the bundling host as the trivial case; replay
  + S-15 notes), §7.5, §11, App B.1; App C S-60 + `module-not-found`;
  plan-m5 D-M5-2 + M5.1 tests; this file. Code: M5.1.
- 2026-08-25 — **S-37 `Procedure` half resolved + spec landed: the
  `Callable`/`Procedure`/`Function` split.** Ruling in the queue entry above.
  One discussions commit: L§4.12 (the list, the umbrella sentence, foreign
  classification, type values not `Callable`; provisional note narrowed to
  spellings), App D spellings, App D.1; App C S-37; plan-m4 D-M4-5 +
  obligation; this file. Code: M4.9.
- 2026-08-25 — **M4.8b — bytes↔string bridging (doodle-rust `e93adb8`).**
  Doodle-callable `encode`/`decode` (provisional intrinsics, §15 "the byte view"):
  `encode(string) → bytes` is the NFC UTF-8 and **cannot fail** (§4.4);
  `decode(bytes) → string` validates UTF-8 + NFC-normalizes, malformed → the new
  **`invalid-utf8`** slug (S-58) with the first bad byte offset in the message; the
  round-trip law `decode(encode(s)) == s` holds. **S-30 resolved:** host `make_string`
  keeps its error-return `ValueError::InvalidUtf8` (no drive to raise into) but now
  carries the same byte position, so the boundary and `decode` tell one story.
  `IntrinsicCtx` gained `alloc_bytes`; both intrinsics register in the native + wasm
  demo registries (parity). Conformance L4.5 (round-trip, decode-normalizes, encode
  byte count, invalid + truncated raises); native 166, wasm 92. O(1) `Bytes` index
  landed in M4.8a. **Carry-forward:** `invalid-utf8` `details: {position, byte}` rides
  the message-rubric / structured-details work (`make_error` is `{}` for all kinds; the
  position is in the message today).

- 2026-08-25 — **M4.8a — graphemes + runtime indexing (doodle-rust `b5f99b5`).**
  `unicode-segmentation` 1.13.3 (**D-M4-3 confirmed UCD 17.0**; all three crates
  aligned and in the compile-time cross-check). `StrObj` gains a lazy grapheme
  memo (`OnceCell` of cluster-start offsets, `unicode::grapheme_offsets`), a pure
  cache excluded from `bytes_allocated` (MD §5 — a heap test pins the invariant).
  `Node::Index` arm gains `List`/`String`/`Bytes` (a string yields a length-one
  grapheme; bytes an `Int`); out-of-range/negative raises the new
  **`index-out-of-range`** slug (S-58, sign-branching message with the no-negative-
  positions hint). `length` (grapheme/element count) + `each` over a string's
  graphemes: provisional intrinsics registered in the native + wasm demo registries
  (parity, E§11). `intrinsic/mod.rs` argument binding split into
  `intrinsic/binding.rs` (length). Native conformance 161 (L4.4 grapheme
  count/iterate, L6.3 index read/out-of-range/negative), wasm 87. **Carry-forwards:**
  `index-out-of-range` `details: {index, length}` rides the message-rubric /
  structured-details work (all kinds are `{}` today); list element assignment
  `xs[i] = v` is a separate pending list-mutation item.

- 2026-08-25 — **M4.7 — string `+`/`*` + AD4 seam renormalization (doodle-rust
  `3bf72ec`).** New `machine/strop.rs` handles `+` (concat) and `*` (repetition)
  off `arith`'s number-only path. The AD4 seam pass is the exported reusable
  `unicode::seam_concat` — renormalizes only the boundary (a's trailing
  non-starter run + anchoring starter; b's leading non-starters + backward-
  composing Hangul V/T jamo), verified equal to whole-string NFC exhaustively
  over an interacting-char alphabet. `+` is String+String only. `*` is symmetric
  (`Int` count either side, S-59): `Float`→type-mismatch, `0`/empty→`""`,
  negative→**`negative-count`** (new S-58 slug), over-heap-limit→`LimitExceeded`
  (the reentry-fault park generalized to `pending_fault`; the R8 interior cap
  stays M4.10). Conformance L4.4 concat/repeat/seam fixtures; native 153, wasm
  79; the huge-`*` fault is a unit test. **Deferred:** the string-churn
  benchmark → its own chunk (queue entry below). **S-30** (host `make_string`
  error model) rides M4.8b.

- 2026-08-25 — **String benchmark suite (doodle-rust `34207e2`; the follow-up
  chunk to M4.7).** `criterion` (default-features off — no plotters/rayon, so
  cargo-deny stays green with no `deny.toml` change) + `benches/strings.rs`:
  `string_churn` (transient concat per iteration — heap turnover/fragmentation
  on the non-moving heap), `combining_mark_append` (the adversarial AD4 O(n²)
  seam case — a lone combining mark appended in a loop), `string_repeat` (a
  large `*`). Driven end to end (parse→resolve→machine); trend-tracking only,
  asserts nothing; `cargo test --workspace` does not run them. **CI trend
  tracking deliberately NOT wired** (a separate scope decision — the plan wants
  it "from M4", but hooking a benchmark into CI is the user's call).

- 2026-08-25 — **S-59 resolved + spec landed: string repetition `*` is
  symmetric.** One discussions commit: L§4.4 (count on either side; `Int`
  only; zero/negative/NFC corners), App D.1; App C S-59; this file. Code:
  M4.7 fixtures.
- 2026-08-25 — **M4.6 — `with`/`parameter` runtime + cancellation cleanup
  (doodle-rust `0cda7a9`).** The M4.5a `WithRestore` producer, end to end
  (new `machine/dynamic.rs`): `parameter` seeds a module cell, `with p = v
  do…end` saves `(cell, old)` on `dyn_stack` and restores on every exit tier;
  cancel-mid-`with` restore is a unit test (accept #5). `Frame.dyn_depth`
  stamps the entry dyn-depth per frame (E§8.2 reads it later). **S-10 runtime
  half:** a valued `break` into a `to`-consumer raises `no-value-destination`
  at the break site (L7.10 `s10-001` via a variable callee, so it stays a
  runtime raise after the deferred static check; + the S-46 native-parity unit
  test); the drive_smoke tripwire graduated into these. **Two ratified extras
  (user, this session):** the **S-5 `with`-tail fix** — `tailcheck` now
  descends a `with` body via a never-falls-through predicate, so
  `fn f() with p do return X end end` (always returns) is accepted, no longer
  mis-flagged fall-off-end; and the **`with`-target static check** — a `with`
  whose target is not a module-level `parameter` (a `let`/`const`/callable/
  type, or undeclared) is `with-target-not-parameter` (§5.5; runtime keeps a
  name-not-defined/used-before-defined backstop). Native conformance 143
  (L5.5 `with-001…007` + `parameter-002`, L7.10 `s10-001`), wasm 69. **Still
  queued:** the S-10 **static** half (valued `break` into a lexically-known
  module-level `to`) rides the front-end resolver extension, alongside the
  loop-half check.
- 2026-08-25 — **S-10 `to`-consumer half resolved + spec landed — S-10 fully
  closed: the S-6 static/runtime split.** Ruling in the queue entry above.
  One discussions commit: L§7.10 (the `to`-consumer sentence; static where
  the callee is a lexically-known module-level `to`, `no-value-destination`
  attributed to the `break` otherwise; bare `break`/valued `continue`
  unaffected), App D.1; App C S-10 (both halves; the resolver follow-up
  extended to the `to`-consumer static check); plan-m4 obligation + M4.6;
  this file. Code: M4.6 runtime half + fixtures.
- 2026-08-24 — **S-58 / D-M4-2 resolved + spec landed: engine errors are one
  built-in `Error` record.** Ruling in the spec-delta queue entry above. One
  discussions commit: L§12.1 (engine-raised values are `Error(kind, message,
  details)`; forgeable; provenance in the trace), §4.12 + App D spellings
  (`Error` joins the type values), App D.1; E§9 + §4.3 (`make_error`) + App
  B.1; App C S-58 (slug catalog); plan-m4 D-M4-2; this file. Code: M4.5b.
- 2026-08-24 — **S-29 resolved + spec landed: record keys need transitively
  immutable content.** Ruling in the spec-delta queue entry above. One
  discussions commit: L§4.8 (the principle + the operational rule + the
  field-naming raise), §9.5 (value records hashable; ref records not by
  default; explicit `Hashable` opt-in), §15 hook 2 (record default; total
  without cycle detection), App D.1; App C S-29; plan-m4 D-M4-1 +
  obligation; this file. Code lands with M4.4.
- 2026-08-02 — **E§3.3 post-raise state resolved + spec landed: a distinct
  terminal `raised` state.** Full ruling in the M2a.3a discovered-delta
  entry above (now resolved). One discussions commit: E§3.3 (the `raised`
  state + the outcome↔state correspondence + the post-mortem observation
  surface + nested-drive scoping), E§9 (outermost-boundary sentence),
  E App B.1; plan-m2b obligation + this file. M2b.3 implements
  (`InstanceState::Raised` replaces the M2a.3a `Faulted` provisional).
- 2026-08-01 — **S-43 resolved + spec landed: namespace-seed, shadowable
  intrinsics.** Full ruling in the spec-delta queue entry above. One
  discussions commit: E§5.5 (provisional pre-module intrinsic
  registration — before-first-`load`, loud late/duplicate errors,
  registration order = replay identity, read-only seeding behind module
  globals, M5 no-observable-change retirement); L§11.4 note (host-injected
  provisional globals — the language still builds no names in); App C
  S-43 (full rationale + rejected alternatives + the parked M5
  shadowing-warning question; folds the M2a.5 type-value provisional
  under it); plan-m2b obligation + this file. M2b.2 implements.
- 2026-07-29 — **S-10 (loop half) resolved + S-9 landed: valued exits need
  a receiving destination; exits punch through `with`/`try`.** One
  discussions commit; full ruling in the spec-delta queue entry above.
  L§7.10 rewritten (the "where applicable" vagueness gone; the valued-exit
  static rule; the pass-through paragraph; bullets tightened), §7.8/§7.9
  cross-notes, App D.1 ×2; App C S-9 RESOLVED + S-10 PART-RESOLVED
  (`to`-consumer half open, fresh ask at M2a.6); MD §20.4 status; plan-m2a
  obligations updated (also fixed a duplicated stale fragment in this
  file's Next-up narrative). Code follow-up queued (next front-end
  session): valued-exit resolver check + battery/conformance tests +
  punch-through pinning test.
- 2026-07-28 — **S-56 resolved + spec landed: float arithmetic raises on
  nonfinite results.** Ruling in the spec-delta queue entry above. One
  discussions commit: L§4.2 (**finite-result rule** — raise iff the IEEE
  result isn't finite; widening included; injected nonfinites inert),
  §3.6.2 (overflowing literal = static error), App D.1; E§4.3 note
  (machine arithmetic never reaches the NaN canonicalization);
  machine-design §3 finite-float invariant + change log (v0.2.3); App C
  S-56 resolution + S-12's `0 ** negative` corner closed by subsumption.
  Code follow-ups: M2a.3 (finiteness check + fixtures), front-end
  literal diagnostic.
- 2026-07-28 — **S-28 resolved + spec landed: total/reflexive numeric `==`.**
  Full ruling in the spec-delta queue entry above. One discussions commit:
  L§4.13 (exact-value equality across kinds, `-0.0 == 0.0`, single
  self-equal NaN), §4.8 (first-inserted key retained), §6.6 (reflexivity
  note; NaN ordering raises), §15 hook 2 (hash-coherence law), App D.1;
  E§4.3 (**Floats normative** — one canonical NaN `0x7FF8…`, `-0.0`
  preserved), §11 (NaN canonicalization as a replay requirement), App B.1;
  machine-design §3/§19 + change log (v0.2.2); App C S-28 (full rationale +
  rejected alternatives) + new **S-56** (nonfinite arithmetic results, due
  M2a.3 with S-12).
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
