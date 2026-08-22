# Working Plan: Milestone M3 — WASM Binding + First Public Demo

**Status:** Proposed (awaiting ratification). · **Parent:**
`implementation.md` §5 M3 `[M]` (WASM binding + first public demo).
Conventions as in `plan-m2b.md`/`plan-m2a.md`. · The core strategy is
**already accepted** in `implementation.md` AD6 (browser main-thread
fuel-sliced pump). This file **sequences** M3 into session-sized work items,
pins the spec deltas that gate the pump (**S-40**) and the browser suspend
contract (**S-15/S-23**), and **surfaces the environment/product decisions**
M3 needs that the engine milestones did not.

Decomposition of M3 — the wasm facade, the JS embedding package with the
fuel-sliced pump, a minimal turtle native surface (with animated `forward`
as a suspending capability), the **nested-drive/JS-suspension prototype**
(the milestone's permanent integration testbed, R4/S-15), and the public
demo page + deploy — into ordered, session-sized work items. Each Rust item
lands its tests green and hygiene clean; each JS item lands with a
Node-driven test; and the milestone closes with an accept-criteria walk +
review (the M1/M2a/M2b rhythm).

**This milestone is different in kind from M0–M2b.** Those were pure Rust
engine work under one toolchain. M3 spans **four surfaces** — the Rust
engine (two spec amendments + one prototype-and-decide), the Rust→JS wasm
facade, a JS/TS npm package, and a browser demo page + deploy pipeline. It
introduces a **JS/TS toolchain and web-hosting** the project has not had,
and several **product decisions** (repo placement, hosting, package naming,
editor/grammar, turtle vocabulary, privacy posture, the unbounded-op guard)
that are the user's to make. Those are collected in **§Decisions needed**
and must be resolved before the items that depend on them.

**Design authority.** `engine.md` (E) governs the embedding contract the
facade exposes (§3 lifecycle, §5 foreign functions + native modules, §7
drive loop + slicing, §8 observation, §10 control). `implementation.md`
AD6 governs the **pump strategy**, AD4 the **Unicode-table size profile**,
AD7 the **js/web repo placement**, §6.5 the **size budget**, §6.4 the
**grammar-parity sub-suite**, §3.9/§7.3 the **environment track**, and R4/R8
the **browser-suspend and unbounded-op risks**. A mechanism change needs the
doc revised first (plan §8). Where an item says "per E§N" / "per AD6", that
section governs.

---

## What M3 delivers (from `implementation.md` §5 M3 + AD6 + R4)

- **S-40 bounded-run fuel + `Paused(SliceEnd)`** in the engine (E§7.2/§7.3):
  a per-drive fuel count (in statement safe points) fused into the M2a.9
  counter, and a slice-exhaustion outcome distinct from a host `HostPause`
  request. The amendment the pump is built on.
- **The nested-drive/JS-suspension prototype (R4/S-15) — the milestone's
  permanent integration testbed.** A native block-consumer whose block
  reaches a **suspending capability**, so a suspend arises *inside a
  reentrant nested drive* (E§5.4) — the case a non-blockable JS host cannot
  block on. M2b.5a currently **faults** this; M3 prototypes the two candidate
  resolutions (**forbid-and-fault** vs **suspend-the-outer-drive**) and picks
  the winner to feed E before M7.
- **`doodle-wasm` facade:** the E§3–§8 embedding surface exposed to JS via
  `wasm-bindgen` — create/load, register natives, drive with fuel, resolve
  capabilities, cancel, read `current_position`, the handle boundary, and a
  **`print` output surface** (the transcript channel conformance compares).
- **`@doodle-lang/engine` JS package:** the **fuel-sliced pump** (AD6) —
  ~8 ms slices via an **injectable timer/frame source** (rAF/`setTimeout` in
  the browser, `setImmediate` in Node CI: *the same facade code CI
  certifies*), the stop button checked **between slices**, position sampled
  **once per animation frame**, and **capabilities surfaced as Promises**.
- **A minimal turtle native surface:** `forward`/`right`/`penup`/… as native
  foreign functions/capabilities, with **animated `forward` a suspending
  capability resolved on rAF completion** (E§5.3), plus a **native
  block-consumer** (`repeat`) that carries the S-15 prototype.
- **The public demo page:** CodeMirror editor + a **separate Lezer grammar**
  for highlighting (the engine exposes no AST, L§13) with the **§6.4
  valid/invalid grammar-parity sub-suite**, a canvas, run/stop, a **live
  executing-line highlight** (E§8.1), and a **`print` output surface**.
- **Deploy + gates:** a CI→static-hosting pipeline producing a public URL;
  the **≤ 300 KB brotli** wasm size gate (AD4/§6.5) and the **≤ 2 s
  first-load** budget; and the **conformance subset green through the wasm
  surface** in Node CI (injected timer — the cross-surface determinism gate).

Provisional prelude natives (`print`/`length`/`to_string` as natives until
the M9a stdlib) are documented as provisional where they appear.

## M3 exit criteria (accept, from `implementation.md` §5 M3 + §6.5)

1. A **public URL** where a **spiral program animates on the main thread**
   with the **currently-executing line highlighted**.
2. The **stop button interrupts an infinite loop "instantly"** (between
   slices — E§10.1; the within-operation caveat is R8/§10.3, guarded per
   **§Decisions #9**).
3. The page **stays responsive throughout** (no main-thread jank).
4. **wasm ≤ 300 KB brotli** (§6.5) **and first-load-to-running ≤ 2 s** on a
   mid-range Chromebook (§6.5).
5. The **conformance subset is green through the wasm surface in Node CI**
   (same facade code, injected timer — the cross-surface determinism claim),
   comparing full transcripts **including `print` output**.

Plus (feeding E before M7, not a public-URL criterion): the **S-15
nested-drive-suspend prototype runs**, and its chosen resolution is written
into E (§spec-delta).

## Spec-delta obligations coming due in M3

Resolve each **in the spec** as part of the item that ships it (plan §8:
edit + App C decision-log entry + conformance/test). The starred ones need a
**fresh user ask** when reached.

- **★ S-40 (E§7.2/§7.3/§8.8) — bounded-run fuel + `Paused(SliceEnd)`.**
  Lands with **M3.1**, *before* the pump. Mechanism pinned in AD6; the
  **E-surface shape** (where fuel sits on the drive API; how
  `Paused(SliceEnd)` relates to `HostPause`/`Step`) is a fresh ask.
- **S-20 (E§7.7/§10.2) — step-budget unit is mode-independent.** Rides with
  M3.1: fuel slicing must not change what the step budget counts.
- **★ S-15 (E§5.3/§5.4/§12) — suspend under a nested drive on a non-blockable
  host. RULED (user, 2026-08-20; §Decisions #2): forbid-and-fault is the
  baseline** — the M2b.5a stub made deliberate (distinct fault type,
  S-16/S-46-family callback contract, the parity control test);
  suspend-the-outer-drive is characterized in E as the deferred M7
  C-ABI-yield extension (a compatible loosening). **M3.3 DONE** — landed the
  `NestedSuspend` engine fault + tests, the E§5.4/§7.6/§7.2 edits, App C S-15,
  and the MD §14/§19 amendments.
- **S-23 (E§10.1) — cancellation robustness, the browser-reachable slices.**
  Comes due because the stop button is accept #2: **cancel of a `Suspended`
  instance discards the pending capability request; a late `resolve` after
  cancel errors** (stop mid-animated-`forward`). Lands with **M3.6** (turtle
  capability) + tested. The **cross-thread carve-out** stays out of scope —
  the pump is single-threaded and checks the flag between slices (no
  cross-thread cancel needed for M3); note that as the M3 boundary.
- **S-16 (E§5.4/§7.6) — abandoned nested drive = host fault.** Re-confirm
  through the wasm surface, but its wording is **contingent on the S-15
  outcome** (the two halves of the browser nested-suspend contract) — defer
  "no new spec text" until M3.3 lands.
- **S-17 (E§7.5/§8) — observation while Suspended.** Re-confirm holds through
  the facade (position readable at a suspend); no new text expected.
- **Turtle = Doodle code, prepended (E§5.5/§13) — RESOLVED (user, 2026-08-20,
  §Decisions #6).** The ratified "minimal native-module lookup" is realized as:
  the turtle is a **Doodle library prepended** to the user program (one module),
  and the *real* native-module/`use` system lands at **M5**. The only host
  surface is the three irreducible **platform primitives** (E§13:
  `draw_line`/`set_turtle`/`clear`, registered like the S-43 provisional
  natives). Provisional natives grow by **`sin`/`cos`** (the turtle's trig,
  until the M9a stdlib) — documented as provisional where they appear.

## Decisions needed (the user's calls — resolve before the dependent item)

Recommended option first; each blocks only the item(s) noted.

1. **js/web repo placement (AD7). RULED (user, 2026-08-21): a new sibling
   submodule `doodle-web`** under `doodle-lang/workspace` for **all** JS/TS +
   web — the `@doodle-lang/engine` pump/package **and** the demo — keeping
   `doodle-rust` **pure Rust** (the `doodle-wasm` crate that compiles to wasm
   stays in `doodle-rust`; `doodle-web` consumes its wasm-bindgen artifact).
   The user weighed this against the hybrid (pump in `doodle-rust`) and chose
   repo purity + independent deploy/npm cadence over a single-repo determinism
   gate. **Consequence:** the **conformance-through-wasm gate spans repos**
   (wasm + fixtures in `doodle-rust`, pump in `doodle-web`); run it as a
   **workspace-superrepo CI job** that builds the wasm from `doodle-rust`,
   pulls fixtures from `doodle-rust/conformance`, and drives them through the
   `doodle-web` pump in Node. **M3.4** (the Rust `doodle-wasm` facade + size
   gate) is `doodle-rust`-internal and needs no `doodle-web`; **M3.5** creates
   `doodle-web` (repo + workspace submodule + CI) — a new-repo setup step to
   confirm when M3.5 begins. 
2. **★ S-15 nested-drive-suspend resolution. RULED (user, 2026-08-20):
   forbid-and-fault as the pinned baseline; NO second prototype.** The
   M2b.5a behavior becomes deliberate, distinctly-typed, tested; E§5.4
   characterizes suspend-the-outer-drive as the deferred M7 C-ABI-yield
   extension (adopting it later is a pure loosening — today's faults
   become working programs). Riders: the outcome is a **fault, not a
   raise** — a catchable raise would make native-vs-Doodle consumers
   program-distinguishable via rescue; the fault extends the S-46 parity
   stance to suspension. The callback contract joins the S-16/S-46
   family (return promptly, no result, no further drives). A **parity
   control test**: the same block + suspending capability through a
   Doodle-written consumer suspends/resumes fine (pins that the
   limitation is the native frame, nothing else). The R4 "prototype
   both" mitigation is **deliberately revised** — a Rust spike would
   explore idioms the C ABI can't carry (closures, borrowed state), so
   its findings wouldn't transfer; the baseline+extension structure
   keeps the door open to exactly M7, the charter's own deadline. Amend
   R4 + App C S-15's "prototype at M3" wording + MD §19 with the M3.3
   landing. Unblocks **M3.3**.
3. **Public-demo hosting + posture (D-6/D-8).** Where the public URL lives —
   **GitHub Pages** under `doodle-lang` (recommended: CI-native, zero infra),
   Netlify/Cloudflare Pages, or a `doodle-lang.dev` domain — and the posture
   (unlisted vs announced). Blocks **M3.8**.
4. **Privacy/analytics for the public kids' page (D-7).** A public page where
   children run code. Recommended: the source default — **no third-party
   analytics for v0.1**, no PII, strict CSP. Blocks **M3.7/M3.8** (before the
   URL goes live).
5. **npm package identity (D-4). RULED (user, 2026-08-21): `@doodle-lang/engine`**
   — the package name in doodle-web. **No `npm publish` in M3** (real publish M7,
   §4.4); a publish-time guard (`prepublishOnly`) fails closed if the wasm/dist
   artifacts are absent.
6. **Turtle registration mechanism — RESOLVED (user, 2026-08-20): the turtle
   is normal Doodle code, not host magic** (the "no-magic-boundaries"
   philosophy — turtle graphics must be ordinary code, not a special feature).
   The turtle is a **Doodle library** over a *tiny* host boundary; for M3 it is
   **prepended** to the user program (one module — no module system pulled
   forward), with the real native-module/`use` system landing properly at
   **M5**. Only the irreducible canvas/animation primitives cross the boundary
   (below). Blocks **M3.2**.
7. **Turtle vocabulary (v0) + state ownership — RESOLVED (user, 2026-08-20).**
   Commands (all **Doodle**): `forward`/`back`/`right`/`left`/`penup`/`pendown`/
   `pencolor`/`home`/`clear`/`showturtle`/`hideturtle`/`repeat` (a Doodle `to`
   with a block param). **State lives in Doodle** (module-level bindings at M3;
   dynamic parameters / ref records are M4). **Color is parsed in Doodle** —
   named table + hex `#rgb`/`#rrggbb`/`#rrggbbaa`, **RGBA (alpha from day one)**;
   `penup` = pass alpha 0. The **host is a dumb drawing surface** — three
   primitives: `draw_line(x0,y0,x1,y1,r,g,b,a)` (animated suspending capability;
   a=0 = pen-up glide), `set_turtle(x,y,heading,visible)` (instant marker pose +
   show/hide), `clear()`. **`forward` needs trig** (stdlib is M9a), so M3 adds
   **provisional native `sin`/`cos`** to the `print`/`length`/`to_string`
   provisional-natives bucket (superseded at M9a). Blocks **M3.2** and **M3.5**.
8. **`print` output surface in the demo + Node.** Recommended: a **DOM output
   pane** in the demo and a **Node capture buffer** in CI (so transcripts
   compare). Blocks **M3.4** (facade output channel) / **M3.7** (demo pane).
9. **R8 — unbounded single-operation guard for a public tab.** A single
   long op (e.g. a huge `**`) can freeze the tab between safe points. For the
   public kid-facing page, recommended: an **M3 interim guard** — restrict the
   demo subset to exclude unbounded single ops until M4's finer limits, *or*
   a host-side deadman timer. Blocks **M3.7** (demo hardening).
10. **§6.5 budget-revision (conditional).** If the ≤ 300 KB brotli gate is
    threatened, §6.5's ladder is: feature-gate rarely-used Unicode tables →
    lazily fetch Unicode data as a separate artifact → **only then a
    budget-revision decision at M3**. Surfaced now; **triggered only if
    M3.4's measurement breaches the gate** (blocks M3.4/M3.8 if so).

---

## Work items (dependency-ordered)

Each item: **Goal**, **Lands**, **Design refs**, **Tests**, **Review**,
**Depends on**. Sizing is roughly one focused session; the JS/browser items
(M3.5–M3.7) are larger and will split when reached (called out below).

### M3.1 — S-40: bounded-run fuel + `Paused(SliceEnd)` (engine + spec) — **DONE**

**Landed** (doodle-rust `a7b3963`; E§7.2/§7.3 + App C S-40 pinned, user-ruled
2026-08-03): `run_slice`/`resolve_slice(fuel: Option<u64>)` variants;
`PauseReason::SliceEnd`; the fused step-budget/slice-fuel counter (fault vs
resumable pause; the fault wins a tie); slicing never gates execution or
perturbs GC (E§7.7). 4-lens review: determinism CLEAN, 3 folded.

- **Goal.** Give the drive a **bounded-run fuel** budget — the primitive the
  pump slices on.
- **Lands.** A **fuel** input on the drive op (per-call, statement safe
  points), **fused** into the M2a.9 counter as `min(remaining fuel, step
  budget, distance-to-next-stop)` so the hot path stays one branch (AD6).
  Exhausting fuel → **`Paused(SliceEnd)`** (new `PauseReason`, resumable,
  state intact); `HostPause` stays a genuine host request. Step-*budget*
  count stays mode-independent (S-20). **Spec:** E§7.2/§7.3 (+ §8.8) + App C.
- **Design refs.** AD6; E§7.2/§7.3/§7.4, §10.2, §8.8; M2a.9 `FusedCounter`.
- **Tests.** A fuel-bounded drive pauses at `SliceEnd` after N safe points
  and resumes to the **same terminal** as an unbounded run (E§7.7);
  `SliceEnd` is distinct from `Step`/`HostPause`/`StepBudget`; step-budget
  count unchanged by slice size.
- **Review.** Fused-counter correctness (no double-count / off-by-one at the
  fuel↔budget boundary), slice-resumption determinism, the outcome↔state map.
- **Depends on.** M2b (drive surface, fused counter, cancellation).

### M3.2 — Platform primitives + the Doodle turtle library (engine) — **DONE**

**Landed** (doodle-rust `aa6425a` turtle + `13fbdef` the `**` sweep; E§11 amended
for transcendental determinism). The turtle is ordinary Doodle code over three
platform primitives — `draw_line`/`set_turtle`/`clear_canvas`, **all suspending
`to` capabilities** (user-ruled all-capabilities 2026-08-20, superseding the
sync-for-the-instant-ones sketch below: uniform, no new engine machinery, engine
stays turtle-agnostic) — plus provisional `sin`/`cos` natives via the bundled
deterministic `libm`. `doodle/turtle.doodle` holds all state/geometry/color;
colors are named + `0xrrggbb` hex-int + RGB(A) channels (the `#rrggbb` *string*
form waits on string primitives — tracked in `claude-todo.md`). A 4-lens
read-only review folded 9 findings; the headline was a determinism sweep-miss —
`**`'s float path used platform `f64::powf`, now `libm::pow` (E§11).

- **Goal.** The turtle as **normal Doodle code** over a tiny host boundary
  (§Decisions #6/#7) — buildable and testable engine-side before any canvas.
- **Lands.** Two layers:
  1. **Host platform primitives** (E§13), registered like the S-43 natives:
     `draw_line(x0,y0,x1,y1, r,g,b,a)` — a **suspending capability** (the JS
     side animates it, M3.5); `set_turtle(x,y,heading,visible)` and `clear()`
     (shipped as `clear_canvas`). *As shipped, all three are suspending `to`
     capabilities* (see the Landed note); the host applies the instant two
     immediately. Plus **provisional native `sin`/`cos`** (the
     turtle's trig, until the M9a stdlib — like `print`).
  2. **The Doodle turtle library** (`.doodle` source, prepended to the user
     program): module-level state (`x,y,heading,pen_down,r,g,b,a`), the commands
     `forward`/`back`/`right`/`left`/`penup`/`pendown`/`pencolor`/`home`/`clear`/
     `showturtle`/`hideturtle`/`repeat`, and **color parsing in Doodle** (named
     table + hex `#rgb`/`#rrggbb`/`#rrggbbaa`, RGBA). `forward` does the trig,
     glides via `draw_line` (alpha 0 when pen-up); turns/teleports/show-hide use
     `set_turtle`. Coordinate/heading convention (Logo: center origin, 0°=up,
     `right`=CW) is a library detail.
- **Design refs.** E§5.1/§5.2/§5.3, §13 (platform primitives); §Decisions #6/#7;
  the S-43 registry (M2b.2) + capabilities (M2b.4). (No native block-consumer —
  `repeat` is Doodle; the S-15 native case is carried separately by M3.3.)
- **Tests.** Crate-internal, driving the prepended library with **scripted
  capabilities**: a square/spiral program issues the expected `draw_line`/
  `set_turtle` sequence (positions, headings, RGBA); `pencolor` parses named +
  hex + alpha; `penup` → alpha 0; `repeat` runs its block per iteration; the
  animated `draw_line` **suspends** and resumes. All in Doodle, no canvas.
- **Review.** The Doodle library's geometry + color parsing correctness, the
  primitive descriptors, that the host surface stays minimal (no turtle logic
  leaks host-side), determinism of the primitive-call sequence.
- **Depends on.** M2b.2/M2b.4 (natives + capabilities), M3.1 not required.
  **∥ M3.1** (both feed M3.4).

### M3.3 — S-15: nested-drive-suspend across the native boundary (engine + spec) — **DONE**

**Landed** (doodle-rust `4a3418f`; E§5.4/§7.6/§7.2 + App C S-15 + MD §14/§19).
Per §Decisions #2 (user-ruled 2026-08-20, re-confirmed 2026-08-21): **forbid-and-
fault**. A suspending capability reached inside a native block-consumer's reentrant
drive is a new, distinct **`Faulted(NestedSuspend)`** — terminal and deterministic,
distinct from `Internal`. The mechanism analysis was **conclusive** that
"suspend-the-outer-drive" is the same save/resume protocol a C foreign function
needs, so it is **characterized in E as the deferred M7 C-ABI-yield extension, not
speculatively built** (R4's "prototype both" satisfied by analysis). Tests: the
native `each`-block-suspends case faults `NestedSuspend` (terminal); the **parity**
case — a *Doodle* block-consumer whose block suspends — suspends+resolves normally
(what the turtle demo's `repeat` relies on). Multi-lens review folded below.

- **Goal.** Resolve what happens when a **suspending capability is reached
  inside a native block-consumer's reentrant drive** — the milestone's
  permanent testbed (R4), which M2b.5a currently **faults**.
- **Lands.** Per **§Decisions #2**: prototype **both** candidate resolutions
  and land the chosen one — (a) **forbid-and-fault**: make the M2b.5a
  `pending`-inside-nested-drive fault a *deliberate, documented*
  host-contract fault (E§5.4); or (b) **suspend-the-outer-drive**: propagate
  the parked capability request out through the native boundary so the whole
  instance goes `Suspended`, and on `resolve` resume back into the nested
  drive. **Spec:** the winner into E§5.3/§5.4 + App C S-15 (before M7).
- **Design refs.** R4 (implementation.md); S-15; E§5.4/§7.6 (reentrant
  drives), §7.5 (suspend), MD §14. The M2b.5a stub (`invoke_block_inner`
  parks `reentry_fault=Internal` on a nested `pending`).
- **Tests.** A program that reaches a suspending capability from inside a
  native `repeat`/`each` block hits the chosen outcome deterministically
  (fault, or suspend→resolve→resume); replay-stable.
- **Review.** The full multi-lens treatment (this is the risk peak): the
  cross-boundary suspend/resume soundness or the fault's terminality; GC
  roots across the parked request; determinism.
- **Depends on.** M3.2 (the native block-consumer), M2b.4/M2b.5a.

### M3.4 — `doodle-wasm` facade (Rust→JS via wasm-bindgen) + the size gate — **DONE**

**Landed** (doodle-rust `e0dd50d`). The E§3–§8 surface as a **natively-testable
core** (`facade::Session`) + a thin `#[wasm_bindgen]` shell (`DoodleInstance` +
`DriveResult`), keeping `doodle-rust` pure Rust (Decision #1). `demo` (print-only,
conformance parity) and `turtle` (natives + prepended library) configs; drive/
resolve over fuel slices; values cross as **opaque handle `u64` ids**; string-tagged
outcomes; `print` output; `currentSpan`/`preludeBytes`/`source` for the line
highlight; `cancel`. The **size gate is now binding**: `wasm-size.sh` measures the
real wasm-bindgen `_bg.wasm` (bindgen→wasm-opt -Oz→brotli) and the CI job installs a
Cargo.lock-matched `wasm-bindgen-cli`. **Measured 178 KB brotli vs 300 KB — 40%
headroom, so AD4's Unicode tables do NOT breach the gate and the §6.5 ladder /
Decision #10 is not triggered.** Node smoke (`print(1+2)`, `forward(10)→Suspended`)
**deferred to M3.5/doodle-web** (no JS harness in the pure-Rust repo). 3-lens
read-only review: encoding/handle-discipline/gate all correct; 6 nits folded.

- **Goal.** Expose the E§3–§8 embedding surface to JS as a small, stable wasm
  API, and **wire the ≤ 300 KB brotli gate as a failing CI check the moment
  real doodle-core wasm exists** (AD4/§6.5).
- **Lands.** `wasm-bindgen` exports: create/load (+register the turtle
  natives), **`drive(fuel)` → an outcome object**, **`resolve(value|raise)`**,
  **`cancel()`**, **`currentPosition()`**, the **handle boundary**
  (`make_*`/readers) for capability args/results and program values, and the
  **`print` output channel** (a capture surface for Node + the demo). Values
  cross as **opaque handle ids**, never raw pointers. Each Run creates a
  **fresh instance + load** (S-33 reuse out of M3 scope, §Decisions). The
  **size gate CI job** + brotli measurement lands here; if breached, invoke
  the §6.5 ladder (§Decisions #10).
- **Design refs.** E§3.1/§4.2/§7/§8.1; AD4 (Unicode tables ~150–350 KB
  dominate the wasm), AD6, §6.5. Extends the M0 stub.
- **Tests.** wasm builds under the pinned toolchain; a **Node** smoke drives
  `print(1 + 2)` (output captured) and `forward(10)` (→ Suspended); the size
  gate reports brotli bytes and **fails over 300 KB**.
- **Review.** JS-boundary shape (outcome/handle/print encoding), no
  determinism leak (E§11), no raw-pointer/UAF, handle release discipline.
- **Depends on.** M3.1, M3.2, M3.3.

### M3.5 — `@doodle-lang/engine`: the pump + capability Promises + conformance-through-wasm (JS/TS) — **DONE**

**Landed** (doodle-web `042b35e` pump + `9f77ed3` conformance/CI; workspace submodule
`8507e0a`; Decisions #1 + #5 realized). Created the **`doodle-web`** sibling submodule
(public, npm-workspaces monorepo) with **`@doodle-lang/engine`** (Decision #5):
`build-wasm.sh` builds `../doodle-rust`'s wasm via wasm-bindgen (`--target web`, same ESM
Node+browser); the **fuel-sliced pump** drives in ~8 ms slices via an **injectable
scheduler**, surfaces **capabilities as Promises** (handler-throw → Doodle raise, E§7.5),
checks the **stop signal between slices AND after each capability** (so a
capability-suspending loop is cancellable), samples position per slice, and streams
output. TS strict. 3-lens review, 10/11 folded incl. a **MAJOR** stop-button fix (was
inert for the animated-turtle shape). **Chunk 2:** the **conformance-through-wasm gate**
(`conformance.test.mjs`) drives every `mode: run` fixture from doodle-rust through the
wasm surface and matches transcript + raise (message + line:col) against the fixture's
directives, replicating the native `matcher.rs` — **the 4 run fixtures pass bit-identically
through wasm (§4.1 cross-surface determinism)**; and **doodle-web CI** (`.github/
workflows/ci.yml`) is the cross-repo gate — checks out both repos as siblings, builds the
wasm (Cargo.lock-matched `wasm-bindgen-cli`), and runs the pump + conformance tests in
Node. 12 Node tests total. **Deferred:** an O(n²) per-slice `output()` copy (needs a
since-offset method on the Rust facade — demo-scale negligible).

- **Goal.** The JS package that pumps the engine jank-free and surfaces
  capabilities as Promises (AD6) — **and** the cross-surface conformance gate.
- **Lands.** The **pump loop**: `drive(fuel)` in ~8 ms slices via an
  **injectable timer/frame source**; **stop checked between slices**; on
  `Suspended`, dispatch to a **registered async capability handler** and
  `resolve` with its result (a **Promise**); **`currentPosition()` sampled
  once per frame**; TS types. Plus the **conformance-subset-through-wasm Node
  CI job** (injected `setImmediate`, scripted capability handlers,
  transcript+`print` comparison to the native runner) — **wired as a failing
  gate here**, when the pump first exists. (Will split: pump core / capability
  dispatch+Promise / conformance harness.)
- **Design refs.** AD6; E§5.3, §7.3, §10.1; §4.3 (conformance drives public
  surfaces), §4.1 (the injected timer is the only nondeterminism, host-supplied).
- **Tests.** Node drives a program to completion through the pump
  deterministically; stop between slices halts an infinite loop within one
  slice; `SliceEnd`→resume reaches the same terminal as one big drive; the
  conformance subset passes through the facade.
- **Review.** Pump correctness (no lost/duplicated slices; stop latency ≤ one
  slice), capability-Promise **single-in-flight** invariant + reject/raise
  path, same facade code in Node and browser.
- **Depends on.** M3.4; **§Decisions #1/#5** (repo placement, npm identity).

### M3.6 — Turtle rendering + animated `forward` + stop-mid-animation (S-23) (JS + canvas) — **DONE**

**Landed** (doodle-web `a95d026`; spec pin `discussions` this commit). New npm
package **`@doodle-lang/turtle`** (§Decisions #7): the browser half of the Doodle
turtle library. `createTurtleHandlers` is the pump `onCapability` — `draw_line`
(cap 3) **animates** over an injectable frame clock (glides the marker, grows the
pen trail, commits one line on completion, E§5.3); `set_turtle` (4) / `clear_canvas`
(5) apply instantly. Heading/visibility persist across a `forward` (only
`set_turtle` updates them); color channels decode from `bigint` handles. A
**two-layer `DrawingSurface`** (committed lines + a transient marker/in-progress
trail): an interrupted `forward` calls `endStroke(false)`, so the canvas shows only
whole strokes. `turtleToPixel` is the pure center-origin/y-up→pixel map;
`CanvasSurface` is the browser `CanvasRenderingContext2D` renderer (owns the scene,
repaints per frame). **S-23:** `animateForward` races each frame against the shared
stop signal and returns promptly on abort; the pump then cancels → `Faulted(Cancelled)`
at the next safe point — **no engine change needed** (the cancel-of-suspended →
resolve-to-unstick → `Cancelled` mechanism was already correct; the handler uses
the value path). **Spec:** E§10.1 gained the **S-23 reaping/late-resolve pin**. **12
tests** — frame-count determinism, pen-up glide, heading/visibility persistence,
clear; and through **real wasm+pump**: hexagon (library `repeat`/block), a 20-side
spiral end-to-end, named-color RGBA, and **stop-mid-`forward` → `Faulted(Cancelled)`
+ line discarded + late-resolve-errors**. CI builds/typechecks/tests all workspaces.
**5-lens read-only review, 1 MAJOR surfaced and FIXED** — a value-vs-raise
cancellation asymmetry in `resolve_slice` (a host raise racing the stop button
escaped cancellation, yielding `Raised` where the value path faulted `Cancelled`).
Per the user's disposition ("fix the engine now"), doodle-rust `6ab0927` makes a
pending cancel discard the resolution and fault `Cancelled` uniformly (E§10.1 S-23
pin updated). M3.6's turtle path was already correct (handler returns normally on
abort → value path).

- **Goal.** Draw the turtle's path and animate `forward` on rAF, and get
  **stop during an in-flight `forward` capability** right (S-23).
- **Lands.** A **canvas renderer** owning turtle state (§Decisions #7);
  **capability handlers** — `forward(n)` **animates** over rAF and resolves
  on completion (E§5.3), sync ones apply instantly; color per §Decisions #7;
  `clear`/`home`. **S-23:** a **stop during an animated `forward`** cancels
  the drive, **discards the pending capability request**, and a **late
  `resolve` after cancel errors** — landed + spec'd (E§10.1). The **spiral**
  renders end-to-end.
- **Design refs.** E§5.3 (rAF capability), §10.1 (S-23 cancel-of-Suspended);
  AD6 (per-frame position); §Decisions #7.
- **Tests.** A **headless** (JSDOM/canvas-mock) test that the handlers
  produce the expected path for a small program (injected frame source →
  deterministic step count); **stop mid-`forward` → `Faulted(Cancelled)`,
  pending request discarded, late resolve errors**.
- **Review.** Turtle-state/rendering correctness, S-23 cancel-race, that
  rendering never re-enters the engine mid-drive.
- **Depends on.** M3.5.

### M3.7 — The demo page: editor + Lezer grammar + run/stop + line highlight + output (JS/HTML)

**Part 1 (Lezer grammar + §6.4 parity gate) — DONE** (doodle-web `c9911bf`;
corpus doodle-rust `96392c0`). New npm package **`@doodle-lang/lezer-doodle`**
(§Decisions: separate pkg + `packages/demo` app): a Lezer grammar for Doodle
(App A + §3), compiled by `lezer-generator` to a **static parse table (CSP-clean,
no runtime `new Function`)**. An **external newline tokenizer + bracket-depth
context tracker** implements §3.2 continuation (significant `newline` at depth 0,
skipped `newlineBracketed` in brackets; structural `nl` after operators/commas).
Engine-mirrored hard cases: **S-4 do-attachment** (a no-block `headerExpr`
ladder), the **two-word `else if` elif** (a no-leading-`if` `nc*` ladder; `else
if` adjacent = elif, `else`+sep+`if` = nested), separator-less protocol/implement
members, docstring-only record body. The **§6.4 parity gate** is a shared corpus
of **20** `mode: static, stage: parse` fixtures (`doodle-rust lang/LA/gp-*`,
engine-verified) that the Lezer grammar must classify identically (`parity.test.mjs`,
cross-repo). A **3-lens read-only review + a ~50-program Lezer-vs-engine divergence
harness** found and fixed **3 false-positives** (unary continuation, member
separators, record body) pre-land. Documented CFG omissions (engine-only checks):
lvalue validity, module-level placement, positional-before-keyword arg order,
docstring placement. Known gaps (corpus excludes): block PARAMETERS `do (x,y)`,
string-internal validation (interpolation/escape/margin), non-ASCII identifiers.
**Next parts:** the CodeMirror 6 editor + `LanguageSupport`/highlighting, then
wiring (run/stop + canvas + `packages/demo`), then line-highlight + output pane —
and **Decision #9 (R8 guard)** before the wiring/hardening (still open).

- **Goal.** The page a user visits — write, run, watch it draw, executing
  line highlighted, stop working, `print` output shown.
- **Lands.** A **CodeMirror 6** editor + a **Lezer grammar for Doodle**
  (separate from the engine, §3.9) with the **§6.4 grammar-parity sub-suite**
  (a small valid/invalid corpus the Lezer grammar and the engine parser must
  classify identically); the canvas (M3.6); **run/stop** wired to the pump;
  a **live line highlight** from the per-frame `currentPosition()` (engine
  gives a byte span, the page maps span→line as diagnostics do); a **`print`
  output pane** (§Decisions #8); a "still-running…" affordance; the **R8
  guard** (§Decisions #9). Strict CSP, no `eval` (§3.9). (Will split:
  editor+grammar / wiring / highlight+output.)
- **Design refs.** §3.9, §6.4; E§8.1 (position = span, host renders); §3.9
  web posture.
- **Tests.** A Playwright/JSDOM test: Run drives a program and Stop halts a
  loop; the highlight advances during a paced program; the §6.4 sub-suite
  passes (grammar↔parser agree valid/invalid).
- **Review.** UX-contract (stop always works, no frozen tab), span→line
  mapping, CSP, the R8 guard, Lezer↔engine parity.
- **Depends on.** M3.6.

### M3.8 — Deploy pipeline + public URL + posture gates (CI/infra)

- **Goal.** Turn the page into a **public URL** with the privacy/hosting
  posture pinned.
- **Lands.** A **CI→static-hosting** deploy (§Decisions #3) producing the URL
  on push to `main`; the **≤ 2 s first-load** budget checked; the **D-7
  privacy posture** enforced (no third-party analytics/PII; CSP) before the
  URL goes live; npm publish **dry-run only** (§Decisions #5). (The size and
  conformance gates already fail CI from M3.4/M3.5.)
- **Design refs.** §4.4 (release engineering), §6.5 (≤ 2 s), D-6/D-7/D-8.
- **Tests.** CI green on a fresh clone through build→gates→deploy; the
  deployed URL serves the page; a first-load timing check.
- **Review.** Deploy secrets/permissions scoped minimally, reproducible
  build, the privacy posture actually enforced.
- **Depends on.** M3.7; **user-hooked** — deploy automation is a scope call
  (CLAUDE.md "stay within asked scope"): confirm before wiring the deploy.

### M3.9 — M3 exit review + accept-criteria walk

- **Goal.** The milestone gate.
- **Lands.** A walk of accept #1–#5 on the deployed URL (spiral animates,
  line highlights, stop instant, responsive, **≤ 300 KB brotli + ≤ 2 s
  load**, conformance-through-wasm green incl. `print`) + the **S-15 prototype
  outcome recorded in E**; a **multi-lens review** (slicing determinism, the
  JS-boundary contract, pump/stop + S-23, the S-15 resolution, size).
- **Depends on.** M3.8 (and M3.3's spec landing).

---

## Notes on ordering and risk

- **Critical path:** M3.1 **∥** M3.2 → M3.3 → M3.4 → M3.5 → M3.6 → M3.7 →
  M3.8 → M3.9. (M3.1 fuel and M3.2 turtle-registration are independent; both
  feed the facade M3.4.) The JS items (M3.5–M3.7) are the long pole.
- **Risk peak: S-15 (M3.3).** The nested-drive-suspend case is the
  milestone's named hard problem (R4) and the current engine **faults** it
  (M2b.5a). Its resolution is a **fresh design decision** (§Decisions #2)
  feeding E before the M7 C-ABI freeze — not a re-confirm. Give it the full
  review treatment.
- **Risk: pump ↔ capability ↔ rAF (M3.5/M3.6).** The first capability
  round-trip through JS async; nail the **single-in-flight-request**
  invariant, **stop-latency ≤ one slice**, and the **S-23** stop-mid-suspend
  race.
- **Size risk is the ENGINE core, not the JS bundle (AD4/§6.5).** The
  ≤ 300 KB brotli gate is on the **wasm**, dominated by AD4's Unicode tables
  (~150–350 KB) that have shipped in doodle-core since M1 — measure at
  **M3.4**, and if breached invoke the §6.5 ladder (feature-gate → lazy-fetch
  → **budget-revision decision at M3**, §Decisions #10). Separately, the **JS
  bundle** (CodeMirror + Lezer + facade) is measured against the **≤ 2 s
  first-load** budget (§Decisions #3 keeps it framework-light) — a distinct
  budget from the wasm gate.
- **R8 — a public kid-facing tab.** A single unbounded op (e.g. `**`) can
  freeze the tab between safe points; the M3 demo needs an interim guard
  (§Decisions #9) since M4's finer limits are not yet in.
- **New toolchain + repo placement (AD7).** M3 adds a **JS/TS build + web
  hosting + a Lezer grammar** the repo lacks; resolve **where they live**
  (§Decisions #1) first, and keep the facade code identical between Node CI
  and browser (AD6) so CI actually certifies the demo.
- **Scope realism.** This is the largest, most cross-cutting milestone and
  the **first outside pure Rust**. The `[M]`-size JS items M3.5–M3.7 will
  each likely split into 2–3 sessions when reached — sequencing, not
  scope-cut. The debugger *panels*, REPL, and richer IDE UX are **M6/M9b**
  (§7.3); only the line highlight + `print` pane ship now.
- **Determinism through wasm (§4.1).** The conformance-through-wasm CI job
  (M3.5) is the load-bearing check that the facade preserves engine
  determinism; the injected timer keeps it reproducible. Cross-surface
  record/replay (browser↔CLI) is **M8**, not M3.
