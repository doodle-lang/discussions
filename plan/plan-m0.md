# Working Plan: Milestone M0 — Scaffolding

**Status:** Draft — submitted for user ratification 2026-07-10 ·
**Parent:** `implementation.md` §5 M0 `[S]`

Decomposition of M0 into session-sized work items. Each item is one
focused agent session or less, with mechanical acceptance. Item IDs
(`M0.n`) are referenced from `../claude-todo.md`; update the status
markers here as items land.

**Ratification note.** Once the user ratifies this plan, the CI wiring
scheduled in its work items is in scope (it is requested here, so
CLAUDE.md's "automation hookup is the user's call" is satisfied by that
ratification); anything beyond what an item states still needs a fresh
ask.

**M0 exit criteria (from the plan):** the pipeline skeleton compiles on
all platforms and a hard-coded AST executes one statement through the
drive API; a placeholder wasm build passes the size gate.

---

## Work items

### M0.1 — Repos, workspace, hygiene, CI, license — **DONE 2026-07-10**

`doodle-lang/workspace` (+ CLAUDE.md), `doodle-lang/doodle-rust` (Cargo
workspace, pinned toolchain 1.97.0, `crates/doodle-core` stub),
`scripts/hygiene/` (6 checks) + dynamic-matrix hygiene CI (green), MIT
license everywhere. Tools installed (rustup/rustfmt/clippy/wasm32 target,
cargo-deny, cbindgen, binaryen, cargo-insta, cargo-fuzz).

### M0.2 — Build/test CI workflow — **TODO**

`.github/workflows/test.yml`: `cargo test --workspace` on
ubuntu/macos/windows; `cargo check --workspace --target
wasm32-unknown-unknown` on ubuntu. Toolchain from `rust-toolchain.toml`
(auto-installed by rustup on the runners). Deliberate deferrals (AD8
completeness): *Node-executed* wasm tests start at M3 with the JS facade;
a Miri job is deferred to M2a, when doodle-core first has nontrivial
code.
*Accept:* workflow green on all four jobs for the current tree.

### M0.3 — Core pipeline skeleton — **TODO**

In `doodle-core`, the module skeleton the front end and machine will fill
in: `span` (Span, ModuleId), `diag` (Diagnostic, LoadError shell), `ast`
(Node arena shell + NodeId), `machine` (Instance shell, `Value` per
machine-design §3, result-register/step scaffolding), `drive` (Outcome
enum per E§7.2, a `run()` that can execute a hard-coded one-statement
AST). No parsing yet. Everything `#[doc]`-ed (missing_docs is warn=deny
via clippy). Gate clarification: machine-design's **§3 `Value` shape is
usable for this scaffolding** — the M2a gate ("no machine-core code
before acceptance") covers the *mechanisms* (frames, unwinding, GC), not
this enum; if the user's machine-design review changes §3, this shell
churns and that is acceptable.
*Accept:* a unit test constructs a hard-coded AST for one statement (e.g.
an integer literal expression statement), drives it to
`Completed`, and observes the result value through the public API — the
plan's M0 acceptance, mechanically checkable. Hygiene green.

### M0.4 — Conformance test file format (mini-spec below) + runner skeleton — **TODO**

Implement `tools/conformance-runner` (workspace member) per §"Conformance
file format v0" below: discovers `conformance/**/*.doodle`, parses
directives, executes against `doodle-core`, reports per-test
PASS/FAIL/SKIP + a clause-coverage summary. Land the format text as
`conformance/README.md` in doodle-rust (source of truth moves there; this
doc keeps the rationale).

**M0 pass policy (so the CI job is green from day one):** `doodle-core`
exposes a staged entry point that reports which stages
(lex/parse/full/run) are implemented; a test whose required stage is
unimplemented is **SKIP**, and the runner exits 0 when there are no
unexpected results — FAIL is reserved for genuine expectation mismatches.
At M0 the core implements no stages, so every test SKIPs and the job is
green; the strict no-skips gate over covered directories turns on at
M1.15 (and fully at M10).
*Accept:* runner runs example tests (SKIP at M0 per the policy); output
includes the overall `=== N passed, N failed, N skipped ===` line; wired
as a CI job (in scope per this plan's ratification note), green.

### M0.5 — wasm hello-world + size gate — **TODO**

`crates/doodle-wasm` stub (wasm-bindgen, exports one function calling
`doodle_core::version()`); `scripts/wasm-size.sh` building
`--release` + `wasm-opt -Oz` + brotli, comparing against the budget
(300 KB brotli, plan §6.5) with the current size printed;
`.github/workflows/wasm-size.yml` (or a job in test.yml) running it.
*Accept:* CI job green; script prints size and budget; deliberately
breaking the budget (locally, with a test blob) makes it fail.

### M0.6 — capi hello-world — **TODO**

`crates/doodle-capi` stub: one `extern "C"` function
(`doodle_version()`), cbindgen generating `include/doodle.h` (committed;
CI check that the committed header is current), and a 20-line C smoke
program in `examples/c-host/` compiled+run in CI (ubuntu).
*Accept:* C smoke test green in CI; regenerating the header produces no
diff.

### M0.7 — insta + fuzz plumbing — **TODO**

`insta` dev-dependency + one snapshot test (e.g. Debug format of the M0.3
hard-coded AST) establishing snapshot conventions
(`crates/doodle-core/tests/snapshots/` layout). `fuzz/` directory with a
cargo-fuzz target stub over a placeholder function (real lexer/parser
targets are M1). Not wired into CI yet (fuzz CI comes with M1 targets).
*Accept:* `cargo insta test` green; `cargo fuzz build` succeeds.

### M0.8 — Contributor docs + spec-delta tracker conventions — **TODO**

`CONTRIBUTING.md` in doodle-rust (build/test/hygiene how-to, the
don't-game-hygiene rule, link to workspace CLAUDE.md, and the **review
policy** the M0 scope requires: what agent review suffices for vs. what
needs user sign-off — this also answers M1.13's reviewer question);
GitHub issue templates in **discussions** (spec-delta template: L/E
section, S-number, provisional choice, resolve-by milestone) and in
**doodle-rust** (bug template referencing the Bug Discovery Protocol).
Decision recorded here: **spec-delta issues live on the `discussions`
tracker** (they are spec work); implementation bugs on `doodle-rust`.
**Blocker needing the user:** issues are currently *disabled* on all
doodle-lang repos — enabling them is a repo-settings action the user must
take or explicitly delegate. Until then, `implementation.md` Appendix C
is the spec-delta tracker of record.
*Accept:* issues enabled (user action); templates render on GitHub's
new-issue pages.

### M0.9 — M0 exit review — **TODO**

Walk the plan's M0 acceptance; update `claude-todo.md` and this file's
status markers; confirm hygiene + all CI workflows green; note any
discovered spec deltas.

**Suggested order:** M0.2 → M0.3 → (M0.4, M0.5, M0.6, M0.7 in any order,
parallelizable) → M0.8 → M0.9. M0.3 blocks M0.4/M0.5/M0.6 only in that
they link against `doodle-core`'s public API.

---

## Conformance file format v0 (mini-spec)

The reviewable draft for M0.4; lands as `conformance/README.md`. Scope:
**language tests** (`.doodle` files). Engine drive-script tests (JSON;
plan §4.3) are spec'd when the drive layer lands (M2b) — a placeholder
section marks the intent.

### Layout and naming

```
conformance/
  v0.1/
    lang/
      L3.2/sep-001_semicolons.doodle
      L6.5/arith-003_int_div_float.doodle
      ...
```

One file per test (multi-module tests get a directory form, spec'd at M5
when imports land). Path encodes the primary clause; the filename is
`<topic>-<seq>_<slug>.doodle`. The **test id** is
`<clause>-<topic>-<seq>` (e.g. `L6.5-arith-003`), unique across the suite.

### Directives

Directives are ordinary Doodle comments beginning `#!` at the top of the
file (before any code; order: header directives, then expectations):

```
#! clause: L6.5            (required; may repeat for secondary clauses)
#! mode: run               (run | static; default run)
#! stage: full             (lex | parse | full; static-mode only; default full)
```

Directive recognition: a directive is `#!` followed by a **space**
(`#! …`); `#!/…` is *not* a directive — it is an ordinary comment, so
shebang lines (L§3.3) remain testable. Directives may only appear before
the first non-comment line.

The `stage:` directive exists so front-end work items can land genuinely
green conformance tests before the whole pipeline exists: `stage: lex`
tests only tokenize (expected errors are lexical), `stage: parse` tests
lex+parse, `stage: full` (default) runs the resolver too. A test SKIPs
when the engine reports its stage unimplemented (M0.4 pass policy).

**`mode: static`** — the test is only loaded (lex/parse/resolve); it
expects either success (no expectation directives) or specific static
errors:

```
#! expect-static-error: <substring> @ <line>:<col>
#! expect-warning: <substring> @ <line>:<col>
```

Every listed error must be reported at the given position (line/col are
1-based, in the NFC'd source per S-1), and no unlisted **errors** may
occur; matching is an **order-insensitive set match** on (substring,
position) — positions disambiguate duplicates. Warnings (e.g. L§5.1
shadowing): every listed warning must occur; *unlisted* warnings never
fail a test (so success-expecting tests are not brittle against new
lints). These tests are runnable from **M1** — before the machine
exists.

**`mode: run`** — the test is executed under the conformance host. The
host defines the capability vocabulary: it registers a `print` capability
whose rendering (each argument via the value's textual rendering,
newline-terminated) it pins in `conformance/README.md`; a declaration
syntax for other scripted capability stubs is a **named placeholder**
(spec'd with the engine drive-script format at M2b), not an implied v0
feature. The expected **transcript** is the ordered list of:

```
#! expect-out: <text>                 (one print line)
#! expect-raise: <substring> @ <line>:<col>   (uncaught error terminating the test)
```

A `run` test passes iff the produced transcript matches the expectations
exactly (count, order, content) and the program terminates (step budget
enforced by the runner; hitting it = FAIL). Runnable from **M2b** (the
host's `print` needs foreign-function registration; raise-only tests may
run from M2a).

### Rules

- Expected text uses **substring** match per event (full-match is too
  brittle against message-wording iteration; positions are exact). The
  error-message *quality* bar is enforced by snapshot tests in
  `doodle-core`, not by conformance.
- Tests are UTF-8, NFC'd by the engine like any source; non-ASCII source
  is encouraged where the clause demands it.
- A test file must be attributable: `clause:` is mandatory and the
  coverage report (M10) is computed from it.
- Frozen suites (`v0.1` after the M10 freeze) are never edited — later
  changes create `v0.2/`.

### Deferred (placeholders)

- **Engine drive scripts** (JSON: directives+resolutions in, expected
  outcome/position/stack stream out) — spec'd at M2b, live in
  `conformance/v0.1/engine/`.
- **Multi-module tests** (directory form with a manifest) — M5.
- **Determinism harness** (`run twice + GC-stress, diff traces`) — runner
  flag, M2a (plan §6.2).
