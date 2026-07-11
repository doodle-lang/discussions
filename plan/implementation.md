# Doodle Implementation Plan

**Status:** Draft · **Version:** 0.1 · **Date:** 2026-07-09

This document is the implementation plan for Doodle: the engine specified in
[`../spec/engine.md`](../spec/engine.md) (**E**), realizing the language
specified in [`../spec/language.md`](../spec/language.md) (**L**), together
with its reference bindings, a bootstrap standard library, a conformance
suite, and the first interactive environment. It is a *plan*, not a
specification: where it pins technical decisions it records rationale,
alternatives, and reversal cost, and it expects to be revised as
implementation teaches us things.

Plan-level decisions and the questions still open are collected in
[§10](#10-open-decisions) and [Appendix C](#appendix-c-spec-reconciliation-backlog).

---

## 1. Goals, non-goals, and deliverables

### 1.1 What "done" means for this plan

This plan covers the path to **v0.1 of the implementation**: a conforming
engine, its two reference bindings, a minimal standard library, a public
browser demo that grows into the alpha IDE, and a frozen conformance suite
that certifies all of it. Concretely, the deliverables:

| # | Deliverable | Form |
|---|---|---|
| D1 | **Engine** (`doodle-core`) | Rust library crate: front end + resumable machine + heap/GC + drive/observation API per E |
| D2 | **C ABI binding** (`doodle-capi`) | `cdylib` + generated `doodle.h` (the artifact E§12 names) + example C host |
| D3 | **WASM/JS binding** (`@doodle-lang/engine`) | wasm + TypeScript facade with async (Promise) drive surface |
| D4 | **CLI host** (`doodle`) | native platform host: `doodle run`, `doodle test`, module resolution over the file system |
| D5 | **Browser demo → alpha IDE** | public web page: editor, canvas, animated turtle, line highlight, stop button; grows into the environment |
| D6 | **Bootstrap standard library** | prelude + well-known protocols + core functions, written in Doodle where possible, over a platform-primitive registry |
| D7 | **Conformance suite v0.1** | versioned, spec-clause-tagged test corpus + runner that certifies through the public bindings |
| D8 | **Spec deltas** | errata and amendments to L and E discovered by implementation, plus a stdlib-spec skeleton |

### 1.2 Non-goals for v0.1

Explicitly out of scope (deferred, with the architecture keeping them cheap):
heap snapshots and reverse time-travel (E§11 defers them; AD3 keeps them
nearly free); live edit / hot redefinition (E§8.9); conditional breakpoints;
the full standard-library specification (D6 ships a *bootstrap* library and a
spec skeleton, not the complete adjacent spec); notebooks, publishing,
multi-turtle; any performance work beyond the budgets in §6.5; localization
of diagnostics.

### 1.3 Assumptions

Team of **1–2 experienced systems engineers**; effort estimates use the
sizing legend in Appendix A rather than calendar dates. The specs L and E
are the source of truth; where the implementation must diverge or decide,
the divergence is filed as a spec delta (§8) rather than silently coded.

---

## 2. Architecture decisions

Each decision records the choice, the reasoning, the alternatives rejected,
and what it would cost to reverse. These were stress-tested twice: first by
generating three independent full-architecture proposals (simplicity-first,
web-first, long-horizon) and adjudicating them — the result is
simplicity-first's engine on web-first's schedule with long-horizon's
determinism discipline — and then by an adversarial review pass whose
surviving attacks are folded into the text and backlog below.

### AD1 — Implementation language: Rust

**Choice.** Rust, stable toolchain pinned via `rust-toolchain.toml`. The
core crate is **safe Rust only** (the slab heap needs no `unsafe` given the
index-based design of AD3); `unsafe` is confined to the C ABI crate where
it is inherent.

**Why.** Rust is the only mainstream option that produces *both* reference
bindings from one codebase: `cdylib` + `extern "C"` + cbindgen for the C
ABI (E§12), and `wasm32-unknown-unknown` + wasm-bindgen for a small, no-
bundled-runtime WASM binary — the browser IDE is the flagship host and
binary size is a budget (§6.5). Exhaustive `enum`+`match` fits a language
implementation; the ecosystem supplies the exact tools needed (num-bigint,
unicode crates, cargo-fuzz, insta, criterion, cbindgen, wasm-bindgen).
Determinism (E§2) is enforceable: no ambient GC, no address-dependent
behavior unless we write it.

**Rejected.** *TypeScript* — fastest to a web demo but cannot produce the
C ABI; a second engine implementation later would be a catastrophe for
conformance. *Go* — WASM binaries are huge; the runtime's own scheduler and
GC fight E§10.3's threading rules and E§2's determinism. *C/Zig* — memory
safety burden for a long-lived 1–2 person project; weaker Unicode story.

**Reversal cost.** Total rewrite. This is the one effectively irreversible
decision; everything else below is module-local.

### AD2 — Execution core: a defunctionalized CESK-style AST machine (no bytecode)

**Choice.** Pipeline: lexer → hand-written recursive-descent + Pratt parser
→ AST with a byte span on every node → **resolver** pass (static errors,
lexical slot assignment, tail-position marking, docstring capture) →
execution by an explicit-state machine: a heap-allocated frame stack where
each frame carries its environment, its **continuation stack** (a `Cont`
enum of pending work: evaluate-operand, finish-call, restore-with-binding,
try-handler, statement-sequence position, …), and its tail-iteration
counter. There is **no bytecode stage**.

**Why.** Resumability (E§7.1) forbids evaluating on the host call stack, so
the real choice is *explicit continuation stack over the AST* vs *bytecode
VM* — a naive tree-walker was never an option. The CESK machine wins for
this project because:

- It deletes an entire artifact class — instruction format, jump patching,
  slot/stack-effect modeling, pc→span tables, register→name debug tables —
  each a bug farm that does not pay for itself at kid scale (a
  defunctionalized Rust machine costs tens of nanoseconds per transition;
  2–10 M statements/s native is realistic, against programs of thousands of
  turtle commands).
- **The machine state *is* the observable state.** Every observation E§8
  mandates — current span, frames with named locals and dynamic-parameter
  bindings, tail history — is a direct read of live structures, not a
  reconstruction through debug tables that can drift. (E§8.8's
  eager-vs-lazy local-capture knob is trivially satisfied: frames hold live
  environments, so locals are always current.)
- **Unwinding exists in exactly one mechanism.** `return`/`break`/
  `continue`, `raise`, and cancellation all pop `Cont` entries;
  `WithRestore` and `TryHandler` are just `Cont` variants, so dynamic-
  parameter restoration on *every* exit path (L§5.5) is structural rather
  than a set of coordinated code paths. This localizes the single most
  bug-prone semantic cluster in the language (§9, R2).
- Suspension is "park the pending capability request and return
  `Suspended`"; `resolve()` writes the result register and re-enters the
  loop (E§7.3). Fine-grained (per-subexpression) observation is a runtime
  flag — no recompilation — matching E§8.8.

**Rejected.** *Bytecode VM* — its genuine advantages are raw speed and a
versionable code format; neither is spec-required, and the performance
budget (§6.5) is comfortably met without it. If profiling ever proves
otherwise, the front end (lexer/parser/resolver, ~half the core) is
unchanged by a swap of the execution module, and the conformance suite is
timing-neutral by construction (E§7.7 + the S-20 budget unit), so a
replacement must merely reproduce the statement-granular observation
stream. **Reversal cost: moderate and contained** — the machine module and
the conformance suite are the insurance.

**Tail calls.** The resolver marks tail positions statically per L§8.7
(a call is not tail inside `with`/`try` cleanup regions). A tail call
reuses the frame in place: same frame identity, same logical depth,
incremented tail-iteration counter, evicted callable pushed into the
config-bounded ring buffer (E§8.3). Stable frame identity makes E§8.5's
stepping rules ("StepOver across a tail call does not run away") a frame-
identity comparison rather than a heuristic.

### AD3 — Heap: index-based slabs, deterministic non-moving mark-sweep

**Choice.** Every heap object lives in per-instance slab vectors; every
reference in the Doodle-visible graph is a **u32 index** — no native
pointers. Handles (E§4.2) are generational u64 slots in a root slab, with
debug-build instance-id tagging to catch cross-instance misuse; the
generation check applies **only at the host boundary** — interpreter-
internal references are raw slab indices with no per-access indirection
cost. GC is precise, non-moving mark-and-sweep, **triggered only by
allocation accounting** (never wall-clock), sweeping **in index order** so
free lists — and therefore all future allocations — are identical across
runs.

**Variable-sized payloads.** Slab entries are fixed-size headers;
string/bytes data and list/dict backings are **per-object Rust
allocations** owned by their header, with their byte sizes counted
explicitly in the allocation accounting (so the E§10.2 heap limit covers
payloads, and fragmentation is delegated to the system allocator).
Determinism needs only index identity and allocation-serial identity, not
payload placement. Consequence for the deferred snapshot API: a snapshot is
"serialize the slabs **plus a payload walk**" — still structural and
position-independent, just not a single memcpy. A string-churn heap
benchmark rides with the R3 suite.

**Why.** Three properties fall out. (1) *Determinism is checkable*: with
index-ordered sweep and allocation-count triggering, heaps are index-
identical at the same step of any replay, so "GC-stress vs. no-GC produce
identical traces" becomes a CI gate (§6.2) instead of a hope. (2) *Identity
is deterministic*: `is_same` and any identity-derived behavior use indices/
allocation serials, never addresses (E§2, E§11). (3) *Snapshots stay
cheap*: instance state is structurally serializable, with the foreign-value
table segregated in its own slab segment so the one genuine hazard (host
pointers) is externalized through a host delegate. Safe Rust throughout —
indices, not pointers.

**Dicts.** Insertion-ordered (L§4.8) compact layout: a dense entry vector
in insertion order plus a hash index; hashing uses **fixed-key SipHash**
(deterministic yet HashDoS-resistant); `std::collections::HashMap` with its
randomized hasher is banned from any Doodle-observable path.

**Rejected.** *Rc + cycle collection* — cycles are legal (ref records), and
RC timing leaks into finalizer behavior. *Moving/compacting GC* — better
locality, but breaks index stability; the indirection is already in place
if a deterministic compactor is ever wanted. **Reversal cost: moderate**
(the heap module is internal to `doodle-core`).

### AD4 — Unicode: small pinned crates behind one wrapper module

**Choice.** `unicode-normalization` (NFC), `unicode-segmentation`
(UAX #29 extended grapheme clusters), `unicode-ident` (UAX #31), all pinned
and accessed through a single internal `unicode` wrapper module. The three
crates version independently and have historically shipped skewed UCD
versions, so the pin is **verified per-crate**: the lockfile audit records
the crate-version→UCD-version mapping, and CI runs generated conformance
vectors at the pinned UCD version — `NormalizationTest.txt`,
`GraphemeBreakTest.txt`, **and an XID_Start/XID_Continue vector derived from
`DerivedCoreProperties.txt`** (UAX #31 has no ready-made test file, and
`unicode-ident` exposes no version constant — the vector run *is* the
assertion). Strings are NFC UTF-8 (L§4.4).

**Seam-only concatenation.** Concatenation renormalizes only around the
join — but "around the join" is defined against **true UAX #15
normalization boundaries**, not "back to the last starter": the scan must
extend across trailing non-starter runs (canonical reordering can move
marks across the seam) and must treat Hangul jamo as composition-active
even though they are starters (L+V→LV, LV+T→LVT compose at the seam). The
whole-string UCD vectors cannot catch a wrong seam algorithm, so
**seam-specific conformance cases** (jamo split L|V and LV|T, non-starter
runs, reordering seams, regional-indicator fusion) are part of M4's
acceptance. String building is linear *except* for adversarial unbounded
non-starter runs (`s = s + "\u{301}"` in a loop is legal and quadratic);
the pathological-append benchmark rides with R3. An unobservable
lazily-built grapheme-offset memo makes repeated `s[i]` tolerable while
preserving the documented O(i) bound.

**Why.** ~150–350 KB of tables versus ICU4X's ~1 MB+ data pipeline;
equally pinnable via lockfile + vendoring + CI vectors. The engine reports
`unicode_version` in its identity; replay artifacts record it and the
replay driver refuses mismatched data (E§11, App C S-36). **Explicitly
rejected** for the WASM binding: the browser's `Intl.Segmenter`/
`String.normalize` — browser-varying ICU would break the Unicode pin and
portable replay. ICU4X is the designated successor if the stdlib ever
needs collation. **Reversal cost: low** (one wrapper module).

### AD5 — Name resolution: lexical slots + module-scope binding cells

**Choice.** Two tiers. Locals and closure captures resolve **statically**
in the resolver to environment slots. Every other bare name compiles to a
reference to a **module-scope binding cell**, resolved lazily on first
execution and then cached. A binding cell is tagged with its kind:
`{value | const | parameter-cell | dispatcher | module}`. Wildcard imports
(`import turtle.*`) **alias the exporter's cells** into the importer's
scope (with provenance tags), so `with pen_color = …` in user code rebinds
the *same* dynamic-parameter cell the turtle library reads — which is the
semantics the language needs and, for dynamic parameters, the only
semantics that *can* work: a parameter is not a value, so a snapshot copy
cannot deliver L§5.5's dynamic-extent behavior across a module boundary.
L§11.3's "all importers share one module instance and its state" and
L§11.2's "brings in the same thing you would otherwise reach as `m.x`"
support the aliasing reading generally.

**The alias is read-only.** Aliasing makes cross-module *assignment*
conceivable (`import shapes.circle` then `circle = 5` would write the
exporter's binding from outside — which L§4.11 forbids in spirit and
L§5.3's text nominally permits). The rule adopted, filed as spec delta
S-39: imported names are **live, read-only aliases** of the exporter's
cells; assignment to an imported name is a **static error** (the resolver's
provenance tags carry the needed information); `with`-binding an imported
dynamic parameter is explicitly legal. Provenance tags also implement the
collision rule (S-13): a name supplied by two wildcards is marked
ambiguous; *using* it raises an error naming both source modules; explicit
declarations and selective imports override wildcards.

**Why.** Wildcard imports make full static resolution impossible: the
names `import m.*` supplies — including which of them are dynamic
parameters — are only known once `m` loads (lazily, mid-drive, per E§6).
The cell-with-kind design makes "the compiler must know a name is dynamic"
(L App D.1) true *at the point of execution* without sigils, keeps reads
O(1), and gives protocol members a natural home (dispatcher cells)
including the ambiguity error L§10.3 requires. "Unknown name" checking
beyond this is an **IDE-linter concern, not an engine concern** — the
engine cannot know a wildcard's names before load, but the IDE's tooling
parser can resolve modules eagerly.

### AD6 — Browser main-thread strategy: fuel-sliced pump

**Choice.** The drive API accepts a bounded-run **fuel** count (in
statement safe points) and returns a dedicated **`Paused(SliceEnd)`**
outcome when exhausted. This is an *engine-API amendment* — E§7.3's
directives have no bounded-run parameter today, and overloading
`HostPause` (a host *request* per E§8.8) would make slice exhaustion
indistinguishable from a user pause — so it is filed as spec delta S-40
and lands in E before M3 ships the pump. The JS facade pumps ~8 ms slices
via an **injectable timer/frame source** (`setTimeout`/rAF in the browser,
`setImmediate` in Node CI — the same facade code is what CI certifies),
checks the stop button between slices, and samples the current position
**once per animation frame** for line highlighting — never crossing the JS
boundary per statement. The fuel check is a **fused counter** (min of
remaining fuel, step budget, and distance to the next stop event), keeping
the per-statement safe point a single predictable branch; step-*budget*
accounting stays mode-independent (S-20).

**Why.** The engine runs on the browser main thread without jank and
without a Worker round-trip per capability; the stop button works between
slices (E§10.1; see R8 for the within-operation caveat); animated
`forward` is a suspending capability resolved on rAF completion — the
canonical validation of E§5.3. Even at the wasm performance floor (§6.5),
an 8 ms slice executes thousands of statements, so instant-speed turtle
programs complete in a few slices.

### AD7 — Repo and packaging

**Choice (as resolved by D-1, 2026-07-10).** The implementation lives in
**`doodle-lang/doodle-rust`**, a submodule of the
**`doodle-lang/workspace`** container repo alongside `discussions`
(mirroring the binate workspace structure). The internal layout below
applies to `doodle-rust`; whether `js/`, `web/`, and `stdlib/` ultimately
live inside it or as sibling submodules is decided when they materialize.

```
doodle-rust/
├── crates/
│   ├── doodle-core/      # front end + machine + heap + engine API (safe Rust)
│   ├── doodle-capi/      # C ABI cdylib; cbindgen → include/doodle.h
│   ├── doodle-wasm/      # wasm-bindgen wrapper
│   └── doodle-cli/       # the `doodle` command: run/test/repl, FS module resolver
├── js/
│   ├── engine/           # @doodle-lang/engine TS facade (fuel pump, async drive)
│   └── turtle/           # turtle native module for the browser (rAF capabilities)
├── web/                  # demo page → alpha IDE
├── stdlib/               # the standard library, in Doodle + primitive declarations
├── conformance/          # versioned, spec-clause-tagged test corpus (D7, §6.3)
├── tools/                # conformance runner, replay tools, size/perf audits
└── examples/             # C host example, gallery programs
```

The `discussions` repo remains the home of the specs and design history for
now; whether specs migrate into the monorepo is deferred to the conformance
freeze (§10 D-2). Spec changes continue to land in `discussions` with the
process of §8.

### AD8 — Engineering baseline

CI from M0: native (Linux/macOS/Windows) + `wasm32` (Node) matrix; rustfmt +
clippy (warnings deny); `cargo-deny` (licenses, advisories, duplicate
versions — the pinned-dependency discipline needs teeth); snapshot tests
via `insta` for diagnostics and AST dumps; `cargo-fuzz` targets for
lexer/parser (later: the machine with scripted capability resolutions, and
the replay-artifact decoder); **Miri on `doodle-core`** (Miri cannot cross
a real FFI boundary) with **ASAN/LSAN/UBSan runs of the example C host**
covering the ABI itself; `criterion` benchmarks and a **300 KB brotli**
wasm size gate (twiggy audits, `panic=abort`, capi feature-gated out of
wasm builds) enforced from the first wasm build. Contributor hygiene
(CONTRIBUTING, issue templates including the `spec-delta` template §8
depends on, review policy for a 2-person team) is an M0 work item.
Error-message quality is a reviewed artifact: every diagnostic has a
snapshot test, and a rubric (names the value, points at the span, suggests
the fix — the bar the design discussion set; see also the L§10.3
example) is applied in review.

---

## 3. Component design notes

Condensed design intent per component; each becomes a tracked work area.

**3.1 Lexer.** NFC-normalizes source first (L§3.1; positions refer to
normalized text — App C, S-1). Emits `NEWLINE` tokens conditionally:
suppressed inside brackets and after a fixed continuation-trigger token set
(symbolic binary operators, comma, `and`/`or`/`is`; trailing `-`/`+`
continue — App C, S-2), seeing through trailing comments. Interpolation
lexes in a string↔expression mode stack with per-level brace counting;
triple-quoted strings capture the raw span first, apply margin validation/
stripping on physical lines, then process escapes (App C, S-3).

**3.2 Parser.** Recursive descent, Pratt for expressions with the 9-level
table (L App C); comparisons non-associative by construction with the
guidance diagnostic. Construct headers (`while`/`with`/`if` conditions)
parse in **no-trailing-block mode** so `while f() do … end` attaches the
`do` to the construct, Ruby-style; a block argument in a header requires
parentheses (App C, S-4 — spec amendment). Produces a fully spanned AST
with docstrings captured as metadata.

**3.3 Resolver.** One pass over the AST: declares/uses lexical slots,
flags static errors (duplicate declaration, `const` reassignment,
undeclared or **imported-alias** assignment where lexically determinable
(S-39), chained comparison, `return`/`break`/`continue` placement,
fn-falls-off-end per the determinability rules of App C S-5, surrogate
escapes, module-level-only placement) and the L§5.1 **shadowing warning**,
marks tail positions (excluding `with`/`try` regions), and classifies every
free name as a module-cell reference (AD5).

**3.4 Machine.** The CESK loop of AD2. One `step()` advances the innermost
frame's continuation stack; safe points fire between statements and at
call/return (E§7.4), where the fused fuel counter and stop conditions are
checked. Values use an 8-byte tagged representation with heap indices;
`Int` is a small-int immediate promoted to a heap bignum on overflow
(num-bigint); `Bytes` is a first-class value with O(1) indexing. Built-in
**type values** (`Int`, `Float`, `Number`, `String`, `Bytes`, `Bool`,
`Nil`, `List`, `Dict`, `Procedure`, …) exist as engine-provided values and
back the `is` operator (type cases from M2a; protocol case from M5); their
provisional spellings and the procedure/function question are pinned by
spec delta S-37. The `Void` sentinel marks procedure-call results and non-
value statements; every value-consuming site rejects `Void` with the
L§6.11 diagnostic (App C, S-6). Interpolation desugars in the resolver to
concatenation of literals with calls to **the `Stringable` dispatcher
directly** — a pinned, hidden binding, *not* the lexical name `to_string`,
so user shadowing cannot change interpolation semantics (L§15 hook 1;
noted in S-37) — ordinary calls that can suspend, raise, and be stepped
like any other code. Assignment targets compile as **place chains**: the
base binding is resolved, then `.field`/`[index]` steps navigate without
copying intermediates, so `a.inner.x = 5` mutates in place rather than a
temporary copy of `a.inner` (value-record copies happen on *binding*, not
on place navigation — spec delta S-38).

**3.5 Drive layer.** Implements E§7's outcomes and directives over the
machine (including the S-40 bounded-run fuel amendment); maintains the
**drive stack** for reentrant drives (E§5.4/§7.6) with host-boundary
sentinel frames that non-local exits and cancellation cannot cross without
terminating the nested drive (App C, S-16); parks and resolves capability
requests (arguments as host-owned handles remaining valid until released —
App C, S-17); implements breakpoints via a statement-span index per loaded
module with pending breakpoints for not-yet-loaded modules (App C, S-21);
raise-trap unified across all three raise sources (`raise`, foreign-
function raise, `resolve(Raise)`) at a single "raise at frame F" entry
point (App C, S-18). Enforces all four E§10.2 limits: step budget, heap
limit (bytes, including payloads per AD3), non-tail stack depth, and the
tail-history bound.

**3.6 Module system.** Per-module state machine
`{unloaded, loading, loaded, failed}` with cycle detection surviving
suspension; module top-level runs as ordinary frames (observable, E§6);
native-module registry consulted before the host resolver; `import a.b`
resolution order per App C S-7. A module whose load raises is marked
`failed` and re-importing raises the same way until an environment-level
reload (App C, S-8).

**3.7 Engine API + bindings.** `doodle-core` exposes the abstract contract
(instances, handles, values, structural inspection, foreign registration —
both synchronous foreign functions and suspending capabilities, with full
L§8.3 argument pre-binding including descriptor defaults and block
parameters (S-42) — foreign values with finalizers, drive/observe) as a
Rust API; `doodle-capi` and `doodle-wasm`/JS wrap it. The C header is
generated (cbindgen) and committed; the JS facade owns the fuel pump (AD6)
and surfaces `Suspended` as a Promise-yielding request object. The
conformance runner certifies through **all three surfaces**.

**3.8 Stdlib bootstrap.** A platform-primitive registry (E§13) populated
by each host (CLI: `print`/`read_line`/time/random over the OS; browser:
the same over JS + the turtle module); the prelude is a host-configured
implicit star-import (AD5 makes this ordinary cell aliasing; the
pre-module provisional mechanism is S-43). Well-known protocols
(`Stringable`, `Hashable`, `Iterable`, `Lengthable`) with built-in
implementations registered natively and record defaults; `each`/`map`/
`filter`/`reduce`/`length`/collection ops; **reflection accessors over the
engine's metadata and the L§15 hook-4 core functions** (`type_of`,
`is_same`, `copy`, `assert`); code-point access implemented **in Doodle**
over the byte view (the L§15 no-magic proof — higher string operations
rest on the engine's Unicode primitives, as L§15 hook 3 provides); the
`test` module and `doodle test` runner. The turtle library is a **Doodle
wrapper module** (declaring `parameter pen_color`, `turtle_speed`, …) over
native drawing primitives — native modules cannot themselves declare
parameter cells (E§5.5; question filed as S-44). Each stdlib addition
records its primitive dependencies — this accumulating registry *is* the
seed of the stdlib spec's platform-primitive section.

**3.9 Environment (demo → alpha IDE).** Grows from the M3 demo page:
CodeMirror editor (with a separate Lezer grammar for highlighting — the
engine deliberately exposes no AST, L§13), canvas, run/stop, line
highlight; then the debugger panels (stack with tail-iteration badges,
locals, dynamic bindings), REPL, and error presentation. Web security
posture: strict CSP, no `eval`, replay-artifact size caps once sharing
exists. The REPL needs three engine-spec additions (incremental top-level
evaluation into a live session module; incomplete-input classification in
`LoadError`; module reload) — filed as spec deltas and scheduled in M9b
(App C, S-24…S-26).

---

## 4. Cross-cutting disciplines

**4.1 Determinism (the load-bearing invariant).** All nondeterminism must
cross the capability boundary (E§11). Enforced by: fixed-key hashing and
deterministic collections only (lint-banned otherwise); allocation-
accounting GC with index-ordered sweep (AD3); identity from allocation
serials; float formatting via a fixed algorithm (Ryū) everywhere; and the
**GC-stress gate** — from M2a, CI runs the conformance suite with
collection forced at every allocation and diffs full position/stack/value
traces byte-for-byte against the no-GC run. From M8, cross-surface replay
(record in browser, replay in CLI) joins the gate. The determinism
obligation on *synchronous* foreign functions is a host-contract gap in E
(App C, S-19) — filed for spec amendment.

**4.2 Error-message quality.** A stated language goal, treated as a
deliverable: every diagnostic (static and runtime) is snapshot-tested; a
negative-program corpus is reviewed message-by-message; runtime errors
name the value and operation, point at the span, and suggest the fix
(e.g. the L§10.3 "to make X iterable, implement Iterable for X" pattern).

**4.3 Conformance suite (the project's normative artifact).** Every test
is tagged with the spec clause it witnesses (`L8.7-ptc-003`); a coverage
report maps every normative *must* in L§3–§13 and E§4–§11 to test IDs
before the v0.1 freeze; frozen suite versions are never edited, only
superseded; the runner drives only public surfaces (Rust API, C ABI, wasm)
so a future second implementation can be certified unchanged. Language
tests are `.doodle` sources + expected transcripts (values, output, raised
values, positions); engine tests are drive scripts (directives +
capability resolutions in, expected outcome/position/stack stream out).

**4.4 Versioning and releases.** The engine reports `(engine version,
spec version, unicode_version)`; replay artifacts record all three plus
the capability-identity scheme version; bumping the Unicode pin is a
*semantics* change (grapheme counts are observable) and gets a spec-level
note. Release engineering is scheduled work, not an afterthought: the demo
deploy pipeline ships with M3, npm/crates publish dry-runs by M7, first
tagged publishes at M10 (§10 D-8 for hosting/cadence).

---

## 5. Milestones

Sizes per Appendix A; **parallel-track effort (§7) is included in the
milestone sizes** — the tracks are views over the same work, not
additional budget. Each milestone closes with: its conformance tests
landed and green on all *then-supported* surfaces, its spec deltas filed
(§8), and its determinism gates green. Dependencies are serial except
where noted.

**M0 — Scaffolding.** `[S]`
Workspace, CI matrix, lint/fmt/insta/fuzz/cargo-deny plumbing,
conformance-runner skeleton + test file format, wasm + capi hello-world
builds, size-gate harness, contributor hygiene (CONTRIBUTING, issue
templates incl. `spec-delta`, review policy).
*Accept:* the pipeline skeleton compiles on all platforms and a hard-coded
AST executes one statement through the drive API; a placeholder wasm build
passes the size gate.

**M1 — Front end.** `[M]`
Lexer (NFC, continuation rules, interpolation modes, triple-string
margins, all literals), parser (full grammar incl. records/protocols/
imports/if-try expressions, header no-trailing-block mode), spanned AST,
resolver static errors + shadowing warnings, LoadError diagnostics with
rendered snippets.
*Accept:* every code example in L parses to a golden AST; a ~40-program
broken-syntax corpus produces reviewed, snapshot-tested diagnostics
pointing at correct spans; every static-error class has a position-
asserting negative test; parser fuzzer survives 1 h.

**M2a — Machine core.** `[L]`
The CESK machine (AD2) + slab heap/GC v0 + handles (AD3) over the *demo
subset*: numbers (incl. bignum promotion), bools, nil, minimal strings,
bytes, lists; built-in type values + `is` (type cases); `let`/`const`/
assignment; `if`/`while`/`loop`; operators with strict booleans and Void
enforcement; `to`/`fn`/anonymous `fn` with keyword args and defaults;
blocks with `continue`/`break`/`return` including punch-through; **PTC
with ring buffer + tail counters**; per-statement safe points + fused
fuel; heap and non-tail stack-depth limits.
*Accept:* a 10⁷-iteration tail-recursive loop runs in constant memory with
the counter reading 10⁷ **while deep non-tail recursion trips
`Faulted(LimitExceeded(stack))`**; an unbounded allocation loop trips
`Faulted(LimitExceeded(heap))` at a deterministic step; **GC-stress
determinism gate green** over the M2a corpus; closures capture loop-fresh
bindings correctly.

**M2b — Drive layer.** `[M]`
Drive loop (`Completed`/`Raised`/`Suspended`/`Paused` + `resolve`, all
directives incl. the S-40 bounded-run fuel); suspending capabilities;
synchronous foreign functions with full argument pre-binding + reentrant
drives; **provisional native-intrinsic registration** (global-scope
foreign bindings so `print`/`repeat`-style names exist before the module
system — S-43, superseded at M5); foreign values + finalizers (GC-time and
destroy-time, host-side only, never Doodle-observable); cancellation; step
budget; current position + stack walk.
*Accept:* a scripted `read_line` capability suspends/resolves; cancel
stops an infinite loop at the next statement; a finalizer runs exactly
once per dead foreign value at GC and per live one at destroy, with
GC-stress traces unchanged; a native `each`-like foreign function invokes
a block reentrantly; drive-directive determinism (E§7.7) holds across
directive mixes.

**M3 — WASM binding + first public demo.** `[M]`
`doodle-wasm` + `@doodle-lang/engine` (fuel pump with injectable timer
source, Promise-surfaced capabilities), minimal native-module lookup
(enough to register the turtle module), the browser **turtle native
module** (`forward`/`right`/… with animated forward as a rAF-resolved
suspending capability), demo page: CodeMirror, canvas, run/stop, live line
highlight; demo deploy pipeline (CI → static hosting); 300 KB brotli gate
active; S-40 lands in E. Provisional prelude natives (`print`, `length`,
`to_string` as natives until M9a) documented as such. This page is the
**permanent integration testbed** for the nested-drive/JS-suspension
contract, not a throwaway.
*Accept:* public URL where a spiral program animates on the main thread
with the executing line highlighted; the stop button interrupts an
infinite loop instantly; page stays responsive throughout; wasm ≤ 300 KB
brotli; conformance subset green through the wasm surface in Node CI (same
facade code, injected timer).

**M4 — Language completion I: data, errors, strings.** `[L]`
Dicts (insertion order, fixed-seed hashing); value/ref records with
copy-on-bind semantics and place-chain assignment (S-38); cycle-safe total
structural `==`; `try`/`rescue`/`raise`/re-raise with traces captured at
raise (live frames + elided history); `with`/`parameter` with restoration
on every exit path and per-frame exposure; full string/Unicode:
seam-renormalizing concat per AD4, grapheme length/index/iteration,
canonical `==`, code-point ordering, `Bytes` bridging with round-trip law
and O(1) `Bytes` indexing, interpolation via the pinned Stringable
dispatcher (running against **native placeholder `to_string`/`hash`
hooks**, replaced by protocol dispatch at M5 and stdlib defaults at M9a).
UCD conformance vectors join CI, including the AD4 seam cases.
*Accept:* L§4–§9 + §12 conformance chapters green on native and wasm
**except clauses requiring protocol dispatch or stdlib defaults (those
clauses are carved into M5/M9a acceptance explicitly)**; composed/
decomposed `"café"` equal with `length == 4`; family-emoji and
flag-fusion grapheme cases pass; jamo/non-starter/reordering seam cases
pass; a crafted raise inside nested `with`/`try`/blocks restores every
dynamic binding and shows tail-elided frames in the trace; **cancellation
inside nested `with`/blocks restores dynamic bindings and runs cleanup
before `Faulted(Cancelled)`**; determinism gate green over the full suite.

**M5 — Modules, imports, protocols, prelude.** `[L]`
Module resolver hook; singleton loading with observable, suspendable
top-level driving; circular-import diagnostics naming the cycle; all
import forms with **cell aliasing + provenance/ambiguity + read-only
aliases** (AD5, S-13, S-39); exports; native modules (registration timing
per S-32); protocols: `protocol`/`implement`, single dispatch, defaults,
`extends`, qualified calls, ambiguity errors, `is` with protocol values;
minimal prelude as host-configured star-import replacing the S-43
provisional mechanism; a small **Doodle-side turtle wrapper module**
(declaring `parameter pen_color` over native primitives — the shape the
real stdlib turtle will take, S-44).
*Accept:* a three-module project where user code's `with pen_color = …`
changes drawing done *inside* the imported turtle wrapper (cell-aliasing
proof); assignment to an imported name is rejected statically with a
provenance-naming diagnostic; a deliberate wildcard collision produces the
two-module ambiguity error on use; a module raising at load is `failed`
and re-import re-raises; `import` of a module with a missing platform
primitive fails per E§13; the M4 carve-out clauses (protocol-`is`,
interpolation over real dispatch) go green.

**M6 — Full observation surface + IDE debugger.** `[M]`
Breakpoints (span index, pending, per-iteration refire), `Step`/`StepInto`/
`StepOver`/`StepOut` with tail-aware depth (frame identity), raise-trap
before unwind for all three raise sources, host pause, pure structural
inspection API, per-subexpression observation mode, auxiliary evaluation
(driving `to_string` on a paused instance under a saved/restored debug
context — App C, S-22). IDE panels: call stack with locals, dynamic
bindings, tail-iteration badges, elided-frame history; watch-it-run mode.
*Accept:* scripted debugger-session conformance tests assert exact pause
positions and stack shapes, including a **directive-semantics matrix**
(the same program with a breakpoint and raise-trap runs to completion
under `RunToCompletion` but pauses under `Continue`); in the live IDE,
StepOver on tail-recursive `polygon` stops each iteration at constant
depth with the badge counting; raise-trap shows the pre-unwind stack; a
supervised debug session works end-to-end in the browser.

**M7 — C ABI + native CLI host.** `[M]`
`doodle-capi` with committed generated `doodle.h` (instances, handles,
outcome union, foreign-function descriptors incl. defaults and block
parameters, resolver callback); the `doodle` CLI (run/test; FS module
resolution; print/read_line/time/random primitives); example C host;
npm/crates publish dry-runs.
*Accept:* a ~300-line C program embeds the engine, registers primitives —
including a foreign function with a default parameter and a block
parameter, both binding per L§8.3 — and runs the conformance suite through
the C ABI with traces identical to the wasm surface; **two instances on
two threads run independently with independent deterministic traces, and a
cross-instance handle use is caught in debug builds**; sanitizer runs
(ASAN/LSAN/UBSan on the C host; Miri on `doodle-core`) clean; `doodle run`
executes gallery programs; suite green through **all three surfaces**.

**M8 — Record/replay v1.** `[M]`
Stable capability-identity scheme — (native-module dotted path, member
name, signature hash), versioned — proposed back into E App B.2 *before*
artifacts circulate; recorder (program identity + ordered request/
resolution stream) and replayer in the JS facade and CLI; a **headless
turtle module** (shared geometry/draw-command backend for browser and
CLI); canonical artifact encoding (CBOR) with the decoder added to the
fuzz targets (it parses untrusted shared input); replay-divergence
detector reporting the first divergent step.
*Accept:* a browser turtle session with user input replays on the native
CLI to a byte-identical position/stack/output trace **and a byte-identical
canonical draw-command stream** (pixel comparison is deliberately not the
criterion — rasterization is not part of the deterministic boundary);
cross-surface replay joins the CI determinism gate.

**M9a — Stdlib bootstrap.** `[L]`
The §3.8 library: well-known protocols with built-in + record defaults
(replacing the M4 placeholders), collection/string/math functions
(code-point access written in Doodle over the byte view), reflection
accessors and the hook-4 core functions (`type_of`, `is_same`, `copy`,
`assert`), the `test` module + `doodle test`; the stdlib-spec skeleton
documenting the accumulated platform-primitive registry; L§13 conformance
chapter.
*Accept:* a real turtle program runs on CLI and browser using only stdlib
+ host turtle module; stdlib's own `_test.doodle` files pass under
`doodle test`; code-point access demonstrably implemented in Doodle
(no-magic proof); `help`-style reflection (name, kind, params, location,
docstring) works for user and stdlib callables; the M4/M5 carve-out
clauses all green.

**M9b — REPL + engine additions.** `[M]`
Engine spec deltas S-24…S-26 land in E first (incremental top-level
evaluation into a persistent session module; incomplete-input
classification; module reload), then: terminal REPL in the CLI; browser
REPL pane beside the canvas.
*Accept:* REPL sessions accumulate definitions, echo values via structural
inspection, continue multi-line input, survive errors (drive-again after
top-level `Raised` per S-33), and `reload` picks up an edited module with
documented staleness caveats.

**M10 — Hardening + conformance v0.1 freeze.** `[M]`
Coverage sweep mapping every normative *must* to test IDs; spec-errata
batch resolving every implementation divergence (App C discharge);
24 h fuzz soak (parser + machine with scripted capabilities + replay
decoder) with zero panics/hangs; systematic error-message review against
the rubric; examples gallery; size/perf audit against §6.5 budgets; first
tagged npm/crates publishes; kid-session acceptance.
*Accept:* coverage report complete; frozen `conformance-v0.1` tagged
alongside spec versions; packages published; a supervised kid session
completes the gallery — including recovering from an accidental infinite
loop via stop — with no page reloads.

**Sequencing.** M0→M1→M2a→M2b→M3 is the critical path to the public demo
(~one quarter for two engineers, honestly ±50%, dominated by M2a — the
least certain estimate in the plan). M4 and M5 are partially
parallelizable between two people (data/strings vs. modules/protocols);
M6–M8 serialize behind M5; M9a can begin against M5. Total to the v0.1
freeze: roughly **9–13 engineer-months** (the milestone sizes sum to that
range at Appendix A midpoints, tracks included), dominated by the four
`[L]`s: M2a, M4, M5, M9a.

---

## 6. Testing and quality strategy

**6.1 Test taxonomy.** (a) Conformance corpus (§4.3) — the normative
artifact, spec-clause-tagged, run through all public surfaces. (b) Unit
tests inside `doodle-core` for machinery (heap, unwinder, lexer modes).
(c) Golden/snapshot tests: AST dumps, diagnostics, traces. (d) Property
tests: `string_bytes∘make_string` round-trip, equality/hash coherence,
NFC idempotence, seam-concat vs whole-string renormalization equivalence.
(e) Fuzzing: lexer/parser from M1; machine-with-scripted-capabilities from
M4; replay-artifact decoder from M8; 24 h soaks at M10. (f) Determinism
gates (§6.2). (g) Integration: the demo page (M3+) and C host (M7+) run
scripted sessions in CI. (h) Acceptance: gallery programs + the kid
session (M10).

**6.2 Determinism gates.** GC-stress trace equivalence (from M2a);
double-run trace diff over the full suite (from M4); cross-surface replay
(from M8). Any diff is a release blocker by definition.

**6.3 The conformance suite doubles as the spec's test.** Writing
clause-tagged tests is how App C items get discharged: an ambiguity that
cannot be tested is an ambiguity that must be resolved in the spec first.

**6.4 Tooling parsers.** The IDE's Lezer grammar is *not* the engine's
parser; a conformance sub-suite of tricky syntax keeps the two from
drifting (both must classify the corpus identically as valid/invalid).

**6.5 Budgets.** wasm ≤ 300 KB brotli (gate from M3), with a named
contingency ladder if the gate is threatened: feature-gate rarely-used
Unicode tables, lazily fetch Unicode data as a separate artifact, and only
then a budget-revision decision at M3. First-load-to-running < 2 s on a
mid-range Chromebook. Performance floors — **measured in statement-level
safe points executed per second**, the same unit as the step budget
(S-20) and the fuel counter: ≥ 1 M/s native and ≥ 300 K/s wasm on the
benchmark suite (sanity floors, not targets — kid programs are thousands
of statements); stepping-mode overhead ≤ 2× RunToCompletion. Measured by
`criterion` + CI trend tracking from M4; the AD2 bytecode reversal is the
contingency if floors are missed, and is not expected to be needed.

---

## 7. Parallel tracks

Track effort is *included* in the milestone sizes of §5; tracks are how
the work is organized, not extra budget.

**7.1 Stdlib + stdlib-spec track (from M4).** The bootstrap library (§3.8)
and the accumulating platform-primitive registry, culminating in the
stdlib-spec skeleton at M9a. Every primitive addition names its E§13
justification.

**7.2 Docs/spec track (continuous).** The §8 process; plus keeping L/E
appendices' decision logs current; plus the M3 fuel amendment (S-40), M8
capability-identity (S-36), and M9b REPL amendments (S-24…S-26) to E.

**7.3 Environment track (from M3).** The demo page hardening into the
alpha IDE (M6 debugger, M9b REPL); UX iterations (animation pacing policy,
"still running…" affordances, error presentation) ride this track without
blocking engine milestones. Beta-tier environment features (adaptive
animation speed, inspectable canvas, save-session-as-program, help/doc
tooltips) and later-tier features (time-travel scrubber, replay sharing,
live edit, notebooks, publishing) are explicitly post-v0.1.

---

## 8. Spec-delta process

Implementation *will* find underdetermined and occasionally wrong spec
text; the backlog in Appendix C already lists ~45 items found by analysis
alone. The process: (1) an implementation divergence or forced decision is
recorded as an issue tagged `spec-delta` with the L/E section; (2) the
implementation may proceed with a documented provisional choice; (3) the
delta is resolved in the spec (edit + decision-log entry in the relevant
Appendix) **no later than the close of the milestone that ships the
behavior**; (4) the conformance test citing the clause lands in the same
change. M10's freeze requires the backlog empty.

---

## 9. Risks

**R1 — Determinism leaks (existential for replay).** Any observable
nondeterminism — randomized hashing, GC-visible timing, address-derived
identity, float formatting, iteration shortcuts — silently breaks E§11.
*Mitigation:* AD3's deterministic heap; §4.1's banned-constructs lint and
GC-stress/double-run/cross-surface gates from M2a/M4/M8.

**R2 — The blocks × PTC × `with`/`try` × non-local-exit tarpit.** The
interaction of punch-through `return`, `break`-to-consumer, tail-call
eligibility barriers, dynamic-parameter restoration, and reentrant host
callbacks is where subtle bugs will live — and a wrong behavior frozen
into conformance-v0.1 becomes permanent. *Mitigation:* AD2 localizes all
unwinding in one continuation-popping mechanism; a dedicated pairwise
conformance battery (every exit kind — **including cancellation** — ×
every construct) lands before M10; several App C items (S-9, S-10, S-16)
pin the semantics in the spec first.

**R3 — String-model performance and heap behavior.** NFC-on-construction
and O(i) grapheme indexing could make idiomatic kid code (string building
in loops, `s[i]` scans) sluggish, especially on wasm; string churn also
stresses the non-moving heap. *Mitigation:* seam-only renormalization
(with correctness pinned per AD4), grapheme memos, ASCII fast paths;
string CPU benchmarks, the adversarial combining-mark append case, and a
string-churn heap/fragmentation benchmark in the suite from M4 so
pathologies surface early.

**R4 — Nested-drive suspension on the browser (structural).** A suspending
capability reached *inside* a synchronous foreign callback cannot block on
the JS main thread (E App B.2 defers this; App C S-15). *Mitigation:* the
M3 demo exercises exactly this contract permanently; candidate resolutions
(forbid-and-fault vs. suspend-the-outer-drive with an async callback
protocol) are prototyped there and the winner is written into E before M7
freezes the C ABI's version of the same question.

**R5 — Scope creep in the environment.** The IDE is unbounded product
work that can starve the engine. *Mitigation:* the environment advances
only on its track (§7.3) behind engine milestones; alpha scope is the
E-supported feature list; everything needing new engine API waits for its
spec delta.

**R6 — Solo-maintainer bus factor and estimate variance.** 9–13 engineer-
months across ~13 milestones with one or two people is honest but high-
variance (M2a especially). *Mitigation:* milestone acceptance criteria are
demos + suites, not dates; the M3 public demo creates an early, visible,
morale-sustaining checkpoint; everything after M3 degrades gracefully in
calendar terms without invalidating the plan.

**R7 — Spec-freeze prematurity.** Freezing conformance-v0.1 with latent
wrong semantics locks mistakes in. *Mitigation:* §8's empty-backlog gate;
the M10 coverage sweep; the discussion's own "no-magic/explicitness"
principles as the tiebreaker for delta resolution.

**R8 — Unbounded single operations defeat the stop button.** Cancellation
fires at safe points, but `9 ** 9 ** 9`, `"a" * 10**9`, or one giant
bignum multiply is a *single* operation — as written it freezes the tab
un-stoppably, contradicting the M3 story. *Mitigation:* interior
cost-accounting poll points inside potentially unbounded primitives
(bignum arithmetic, string/list repetition, normalization of huge inputs)
and/or operand-size caps tied to the heap limit, specified alongside S-12
and conformance-tested by cancelling mid-operation (scheduled with M4's
string work).

---

## 10. Open decisions

Decisions this plan needs from the project owner (none block M0–M1 except
D-1):

- **D-1 · Implementation repo.** **RESOLVED (2026-07-10):**
  `doodle-lang/doodle-rust`, a submodule of the `doodle-lang/workspace`
  container repo alongside `discussions` — the binate workspace
  structure, chosen over the originally proposed `doodle-lang/doodle`
  monorepo. All repos are **public** (the user's deliberate choice,
  2026-07-10, superseding the earlier private posture). AD7's internal
  layout applies within `doodle-rust`; sibling-vs-internal placement of
  `js/`/`web/`/`stdlib/` is deferred until they materialize.
- **D-2 · Spec home.** Keep specs in `discussions` through v0.1 and revisit
  at freeze (recommended), or migrate `spec/` into the monorepo at M0.
- **D-3 · License.** **RESOLVED (2026-07-10): MIT**, across all project
  repos (workspace, discussions/specs, doodle-rust), matching the binate
  projects. Crates declare `license = "MIT"`; cargo-deny's
  private-crate exception is dropped.
- **D-4 · Names.** Crate prefix `doodle-*`; npm scope `@doodle-lang`
  (`@doodle` is likely taken); CLI binary `doodle`. Confirm or rename.
- **D-5 · Unicode pin.** Pin v0.1 to a single UCD version across all three
  crates — target **Unicode 16.0**, subject to the per-crate verification
  AD4 requires (the crates version independently; the audit may force
  pinning to the newest version all three actually support). Confirm the
  approach.
- **D-6 · Public-demo posture.** M3 produces a publicly reachable URL —
  fine to be quietly public, or should it stay behind an unlisted link
  until M10?
- **D-7 · Privacy/analytics posture.** A public page where children run
  code needs an explicit stance (COPPA/GDPR-K-adjacent). Recommend: **no
  third-party analytics for v0.1**; document exactly what replay artifacts
  contain (they can embed typed input via `read_line` resolutions) before
  sharing exists at M8.
- **D-8 · Hosting and release cadence.** Static-hosting provider for the
  demo (CI-deployed), docs hosting, and a semver/release-cadence policy
  for the npm/crates packages (first publishes at M10).

---

## Appendix A — Sizing legend

Per engineer, focused work, including tests and spec deltas:
**S** ≈ up to a week · **M** ≈ 1–3 weeks · **L** ≈ 3–6 weeks ·
**XL** > 6 weeks (deliberately absent: anything sized XL was split — this
is why M2 and M9 appear as M2a/M2b and M9a/M9b). Estimates are honest
medians; M2a and M5 carry the widest error bars.

## Appendix B — Decision summary (quick reference)

| ID | Decision | Reversal cost |
|---|---|---|
| AD1 | Rust, pinned stable, safe-only core | total rewrite |
| AD2 | CESK-style explicit-continuation AST machine; no bytecode | moderate, contained to the machine module |
| AD3 | Index-slab heap (per-object payloads, byte-accounted); deterministic mark-sweep; generational handles (boundary-only checks); fixed-key SipHash dicts | moderate, internal |
| AD4 | Small pinned Unicode crates + UCD/seam/ID-property vectors; ICU4X designated successor | low |
| AD5 | Lexical slots + module binding cells; wildcard = live **read-only** cell aliasing + provenance | low-moderate |
| AD6 | Bounded-run fuel (S-40) + `Paused(SliceEnd)`; ~8 ms slices; rAF position sampling; injectable timer | low |
| AD7 | `doodle-lang/doodle` monorepo; specs stay in `discussions` for now | low |
| AD8 | CI matrix, 300 KB brotli gate, fuzz, cargo-deny, sanitizers-on-C-host + Miri-on-core, snapshot diagnostics | — |

## Appendix C — Spec-reconciliation backlog

Underdetermined or problematic spec points that implementation forces;
each is discharged per §8 no later than the milestone shown. (L) = language
spec, (E) = engine spec. Condensed. **This appendix is the spec-delta
record of substance**; GitHub issues on the `discussions` repo (enabled
2026-07-10; templates land with plan-m0 M0.8) track and point at its
entries.

**Front end — resolve by M1.**
S-1 (L§3.1/E§8.1) Source-position units: define positions on NFC-normalized
text, spans in code points (bindings may convert). **[resolved M1.2 — L§3.1 +
App D.1]** ·
S-2 (L§3.2) Exact continuation-trigger token set (trailing `-`/`+`, word
operators, `=`; comments between operator and newline). **[resolved M1.3 —
L§3.2 + App D.1; `=` is NOT a trigger]** ·
S-3 (L§3.6.4) Triple-string margin details: tabs vs spaces (exact-prefix
match), blank-line exemption, margin-before-escapes ordering. ·
S-4 (L§6.4/§7) `do`-attachment in construct headers (`while f() do`):
header expressions parse in no-trailing-block mode. ·
S-5 (L§5.3/§6.11/§8.4) Define the "static where determinable" analysis
boundary so conforming implementations reject the same programs. ·
S-6 (L§6.11, §8.4, §8.5) Void semantics: block/fn whose last statement is
value-less — error at the consuming site, uniformly (amends §8.4's
"raises at runtime" and §8.5's block-value sentence accordingly). ·
S-7 (L§11.2) `import a.b` module-vs-member resolution order (try module
path first, member fallback; native registry precedence). ·
S-27 (L§8.6) Docstring corners: body that is only a string; interpolation
in docstrings (treat as raw text — not executed).

**Core semantics — resolve by M2a/M2b/M4.**
S-9 (L§7.10) `break`/`continue` inside `with` inside a loop: as written,
loop control from a `with` body is impossible — almost certainly
unintended; fix the interaction (and specify the `try`-body case). ·
S-10 (L§7.10) `break`-with-value where the consumer has no value slot
(plain `while`/`loop`, `to` consumers): specify (discard vs error). ·
S-11 (L§6.10/§8.5) State explicitly whether `fn` closures may *mutate*
captured bindings (blocks explicitly may; align or distinguish). ·
S-12 (L§4.2) `**` open corners: `0 ** negative` (division-by-zero
analog?) and resource behavior for huge exponents (ties to R8's interior
poll points). (Int ** negative-Int → Float is already settled by L§4.2.) ·
S-28 (L§4.13) Numeric equality precision: Int/Float comparison beyond 2⁵³
(compare exactly, not via lossy widening); NaN and −0.0 under total
structural equality; hash coherence for `1` vs `1.0`. ·
S-29 (L§4.8/§15) Mutable records as dict keys: pin the behavior (document
stale-hash reality vs. restrict default Hashable to immutable content). ·
S-30 (E§4.3) `make_string` failure mode from the host (error return, not
raise) and the general no-drive-in-progress host-call error model. ·
S-37 (L§4.12/App D.1, L§15 hook 1) Pin built-in type-value spellings and
whether `Procedure` distinguishes procedures from functions (reflection,
`is`, and error messages all need it); pin that interpolation invokes the
Stringable dispatcher directly (immune to lexical shadowing of the name
`to_string`). Resolve by M4. ·
S-38 (L§5.3/§4.14) Lvalue place chains: assignment targets evaluate as
places (no intermediate value-record copies), so `a.inner.x = 5` mutates
`a`; copies happen on binding, not place navigation. Resolve by M4. ·
S-41 (E§3.1) Reconcile the `create(config)` "target Unicode version" field
with the build-time pin: engine reports its pinned version; `create` fails
on an unsupported request (or the field is dropped for the identity
report). Resolve by M2a. ·
S-43 (E§5.5/L§11.4) Provisional pre-module native-intrinsic registration
(global-scope foreign bindings) so core names exist before the module
system; superseded by the prelude star-import at M5 — specify the
mechanism and its retirement. Resolve by M2b. ·
S-45 (L§8.7) A call that passes a block argument is **not** a tail
position: the block references the caller's frame, so the frame cannot be
reused (found during machine design — see `machine-design.md` §11). Amend
L§8.7's non-tail list alongside the with/try barriers. Resolve by M1
(tail marking), consumed by M2a (frame reuse).

**Modules/protocols — resolve by M5.**
S-13 (L§11.2) Wildcard-collision rule: second wildcard supplying an
existing wildcard-bound name marks it ambiguous; use raises naming both
modules; explicit/selective bindings override. ·
S-8 (L§11.3/E§6) Module load-failure state: a module whose top-level
raises is `failed`; re-import re-raises (no silent retry); reload is an
environment-level operation (S-26). ·
S-14 (L§11.1/§11.3) `exports` corners (multiplicity, undeclared names,
position); module-block semantics. ·
S-31 (L§10) Protocol-implementation signature conformance (arity/defaults/
block param must match declaration) and dispatch when the first argument
arrives by keyword. ·
S-32 (E§5.5) Native-module registration timing (before first drive;
mid-run registration deferred). ·
S-39 (L§11.2/§4.11/§5.3/§5.5) Imported names are live, **read-only**
aliases of the exporter's binding cells: reads see the exporter's current
value; assignment to an imported name is a static error; `with`-binding an
imported dynamic parameter is legal (and is the only way cross-module
dynamic parameters can work). ·
S-44 (E§5.5) Whether native modules may declare `parameter` cells, or
whether dynamic parameters always live in Doodle wrapper modules over
native primitives (the plan assumes the latter).

**Engine/drive — resolve by M2b/M3/M6/M7.**
S-15 (E§5.3/§5.4/§12) Suspending capability under a nested drive on a
non-blockable host: prototype at M3, specify before M7. ·
S-16 (E§5.4/§7.6) Abandoned nested drives (callback returns while its
nested drive is Suspended/Paused): define as a host-contract fault. ·
S-17 (E§7.5/§8) Observation while Suspended: capability call sits at an
implicit safe point; request-argument handles are host-owned. ·
S-18 (E§8.7) Raise-trap unified across `raise`/foreign-raise/
`resolve(Raise)` (trap fires before any unwind in all three). ·
S-19 (E§11) Determinism obligation on synchronous foreign functions
(host contract: sync FFs must be deterministic or become capabilities). ·
S-20 (E§7.7/§10.2) Step-budget unit is mode-independent (statement safe
points) regardless of observation granularity. ·
S-21 (E§8.6) Breakpoint mapping details: span index, lines without code,
multiple statements per line, pending breakpoints for unloaded modules,
canonical-id reuse. ·
S-22 (E§8.4) Auxiliary evaluation (host-driven `to_string` on a paused
instance): saved/restored debug context, own small budget, breakpoints and
raise-trap suppressed. ·
S-23 (E§10.1) Cancellation robustness: reserved unwind budget; capability
call during cancel-unwind faults; second cancel = hard abort; cancel of a
Suspended instance discards the pending request; late `resolve` errors.
Also: cross-thread carve-out — cancel/pause are the only thread-safe
operations (atomic flags polled at safe points). ·
S-33 (E§3.3/§7.3) Instance reusability after `Raised`/`Faulted` at top
level (REPL needs drive-again); raising a step budget after
`LimitExceeded`. ·
S-34 (E§8.3) Ring-buffer scoping (instance-global; snapshot into traces at
raise-capture; entries tagged with consuming frame serial). ·
S-35 (E§4.2) Handle `retain` semantics (per-handle count; double-release
and use-after-destroy are contract violations caught in debug builds). ·
S-40 (E§7.2/§7.3/§8.8) Bounded-run fuel: a fuel parameter (in statement
safe points) on the drive operation and a dedicated `Paused(SliceEnd)`
reason, keeping `HostPause` a genuine host request. Resolve by M3. ·
S-42 (E§5.1) Foreign-function descriptor argument binding: how defaults
are represented (value handles vs engine-evaluated) and block-parameter
declaration; conformance-tested through the C ABI. Resolve by M7. ·
S-46 (E§7.2/§5.4) Non-local exits (`break`/`return`) crossing a **native
block-consuming callee**: a nested drive returns a distinguished
`NonLocalExit{kind}` outcome; the foreign callback must return promptly
without a result; the engine resumes the parked unwind at the foreign
call's apply site (a Break targeting that call completes it with the
value; anything else keeps unwinding); a host that returns a value or
raises after `NonLocalExit` faults. Found by the machine-design review
(v0.2 §12 — S-16 covers abandoned drives, not this). Resolve by M2b.

**Environment-driven engine additions — resolve by M9b.**
S-24 (E§3.2-new) Incremental top-level evaluation into a persistent
session module (the REPL API). ·
S-25 (E§3.2) `LoadError` classifies incomplete input distinctly from
syntax errors (multi-line REPL entry). ·
S-26 (E§6-new) Module reload: invalidate + re-drive + rebind semantics
(documented staleness caveats).

**Replay — resolve by M8.**
S-36 (E§7.5/§11/App B.2) Capability-identity scheme: (module path, member
name, signature hash), versioned in the artifact with engine/spec/unicode
versions; replay driver refuses mismatches.
