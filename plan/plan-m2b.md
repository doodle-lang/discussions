# Working Plan: Milestone M2b — Drive Layer

**Status:** Proposed (awaiting ratification). · **Parent:**
`implementation.md` §5 M2b `[M]` (the host/embedding layer). · Conventions
as in `plan-m2a.md`/`plan-m1.md`. · The mechanisms it implements are
**already accepted** in `machine-design.md` v0.2.5 — §14 (suspension + the
drive stack), §12 (`HostBoundary`/`NonLocalExit` park-and-resume), §15
(foreign finalizers), §16 (handles). This file only **sequences** them into
session-sized work items and pins the M2b spec deltas.

Decomposition of M2b (the drive-state machine, foreign functions,
suspending capabilities, reentrant drives, foreign values + finalizers,
cancellation, the minimal observation surface) into ordered, session-sized
work items. Each item lands its doodle-core tests green and hygiene clean,
and gets a minimal read-only adversarial review before it lands (the M1/M2a
rhythm).

**Design authority.** `machine-design.md` governs the *mechanisms*
(§14 drive stack, §12 unwind/`HostBoundary`, §15 GC/finalizers, §16
handles); `engine.md` (E) governs the *embedding contract* (§3 lifecycle,
§4 value model, §5 foreign functions, §7 drive loop, §8 observation, §10
control/limits). Where an item says "per MD §N" or "per E§N", that section
governs; a mechanism change needs the doc revised first.

---

## What M2b delivers (from `implementation.md` §5 M2b)

- **Drive loop:** `Completed`/`Raised`/`Suspended`/`Paused` + `resolve`,
  all directives (the S-40 bounded-run-fuel amendment is flagged below as a
  sequencing question — the backlog lands it in E at **M3**).
- **Suspending capabilities** (E§5.3): a scripted `read_line` suspends and
  resolves.
- **Synchronous foreign functions** (E§5.2) with full argument pre-binding
  (L§8.3) + **reentrant drives** (E§5.4/§7.6): a native `each`-like
  primitive invokes a block reentrantly.
- **Provisional native-intrinsic registration (S-43):** global-scope
  foreign bindings so `print`/`repeat`-style names exist before the module
  system; superseded by the M5 prelude star-import.
- **Foreign values + finalizers** (E§4.5): GC-time (dead) and destroy-time
  (live), host-side only, never Doodle-observable.
- **Cancellation** (E§10.1); the **step budget** (already enforced by the
  M2a.9 fused counter — M2b wires it into the drive surface); **current
  position + stack walk** (E§8.1/§8.2, the minimal observation surface).

## M2b exit criteria (from the plan)

1. A scripted `read_line` capability **suspends** and **resolves** (the
   result becomes the call's value; a `Raise` resolution raises at the call
   site).
2. **Cancel** stops an infinite loop at the next statement →
   `Faulted(Cancelled)`, not catchable.
3. A **finalizer** runs **exactly once** per dead foreign value at GC and
   per live one at destroy, with **GC-stress traces unchanged**.
4. A native **`each`-like** foreign function invokes a block **reentrantly**.
5. **Drive-directive determinism** (E§7.7) holds across directive mixes.

Plus, unblocked by M2b: the SKIP'd `expect-out` conformance run-tests
(e.g. `arith-001`) execute once `print` lands (M2b.2) — the runner's
`needed_capability`/`print` gate lifts.

## Spec-delta obligations coming due in M2b

Resolve each **in the spec** as part of the item that implements it (plan
§8: edit + App C decision-log entry + conformance/test). Flagged here so
none is silently deferred (CLAUDE.md). The starred ones need a **fresh
user ask** when reached (semantics/mechanism not yet ratified).

- **S-43 (E§5.5/L§11.4) — provisional pre-module native-intrinsic
  registration. RESOLVED (user, 2026-08-01; App C + spec landed):
  namespace-seed, shadowable.** Intrinsics register before the first
  `load` (late/duplicate = loud host-API error; order = replay-identity
  input) and seed as read-only global cells behind module globals — a
  user's own declaration shadows them; assignment is already a static
  error (S-39/M1.10b). Folds the M2a.5 type-value BUILTINS seeding under
  the same mechanism; the M5 star-import retires it with no
  program-observable change. **M2b.2** implements.
- **Post-`Raised`/`Faulted` instance state (E§3.3). RESOLVED (user,
  2026-08-02; spec landed): a distinct terminal `raised` state.** Each
  stopping outcome leaves the instance in the same-named state, so
  `state()` alone preserves E§9's raise-vs-fault line; exception + trace
  stay observable post-mortem (stack unwound; §8.7 observes pre-unwind);
  a nested drive's `Raised` changes no state (outermost only);
  cancellation stays `Faulted(Cancelled)`. **M2b.3** implements
  (`InstanceState::Raised` replaces the M2a.3a provisional; tests:
  raised-after-raise, faulted-after-limit).
- **S-46 (E§7.2/§5.4) — non-local exits crossing a native block-consuming
  callee. DIRECTION CONFIRMED (user, 2026-08-02): the MD §12
  `NonLocalExit` mechanism** (`NonLocalExit{kind}` outcome; callback
  returns promptly without a result; engine resumes the parked unwind at
  the foreign call's apply site — a Break targeting that call completes
  it per L§7.10, a `return`/outer break keeps unwinding; value/raise
  after `NonLocalExit` = host-contract fault). **Riders the E edit must
  pin** (lands with **M2b.5** + App C S-46): (1) `continue` never
  crosses — it is the block invocation completing normally
  (`Completed(value?)` to the callback), not a `NonLocalExit`; state the
  three-way symmetry with `Raised` (E§9) once. (2) Multiple crossed
  native frames compose — each callback sees `NonLocalExit` innermost
  outward, unwind resuming at each apply site. (3) **No new drives after
  `NonLocalExit`**: that callback activation may clean up host-side but
  may not re-invoke the block or start nested drives (same
  host-contract-fault family as S-16); Doodle-level cleanup belongs to
  the engine's unwind segments. (4) **Parity is the acceptance bar**: a
  native consumer is program-observably identical to a Doodle-written
  one for every exit kind — and S-46 stays orthogonal to S-10's open
  `to`-consumer half (transport vs value semantics; don't pre-decide it
  in the E text). (5) Replay: `NonLocalExit` and the resumption are
  engine-derived — no new recordable input; one sentence in the edit.
- **S-16 (E§5.4/§7.6) — abandoned nested drives** (callback returns while
  its nested drive is Suspended/Paused): define as a host-contract fault.
  Lands with **M2b.5**.
- **S-42-lite (E§5.1) — foreign-function descriptor argument binding.** How
  defaults and the block parameter are represented in the descriptor. The
  *full* S-42 (C-ABI conformance) is M7; M2b pins only the in-engine
  descriptor shape it uses. Lands with **M2b.2**.
- **S-35 (E§4.2) — handle `retain` semantics** (per-handle count; double-
  release / use-after-destroy = debug-caught contract violations). The
  M2a.11 `HandleTable` already implements the generation check; M2b pins
  the wording as the public boundary API lands (**M2b.1**).
- **S-19 (E§11) — determinism obligation on synchronous foreign functions**
  (host contract: a sync FF must be deterministic or become a capability).
  Note in E with **M2b.2**.
- **S-17 (E§7.5/§8) — observation while Suspended** (a capability call sits
  at an implicit safe point; request-argument handles are host-owned).
  Confirm with **M2b.4**.

## Sequencing decisions (ruled by the user, 2026-08-01)

These were scope-boundary calls the milestone text left ambiguous. Both are
now decided (the recommended options); recorded here so the decomposition
reads as ratified on these points.

1. **The `Step*` directives / breakpoints — M2b vs M6. → RULED: basic Step
   in M2b, the rest M6.** M2b lands the drive-state-machine *plumbing* for
   every directive and a **basic** `Step`/`Continue` (pause at the next
   statement-level safe point; `StepInto/Over/Out` by frame-depth against
   the existing safe points; `Paused(Step)`). Breakpoints, tail-aware
   stepping depth, raise-trap-as-pause, per-subexpression granularity, and
   the inspection panels stay **M6**. This keeps M2b's `Paused` real and
   testable (safe points already exist) without pulling the debugger
   forward. (Governs **M2b.3**.)
2. **S-40 bounded-run fuel + `Paused(SliceEnd)` — M2b vs M3. → RULED: M3
   (deferred).** Scoped **out of M2b**: `HostPause` stays a genuine host
   request, and M2b adds no fuel parameter the demo hasn't shaped yet. S-40
   lands in E at **M3** (the App C backlog and the M3 milestone text). The
   M2b summary line's "incl. the S-40 bounded-run fuel" is thereby a
   forward reference, not an M2b obligation.
3. **S-44 (E§5.5) — may native modules declare `parameter` cells?** Out of
   scope for M2b (its S-43 intrinsics are global-scope foreign *functions*;
   no dynamic `parameter` is needed until M4/M5). No M2b code touches it;
   left for **M5** as the plan assumes. Noted so it isn't forgotten.

---

## Work items (dependency-ordered)

Each item: **Goal**, **Lands**, **Design refs**, **Tests**, **Review**,
**Depends on**. Sizing is one focused session apiece (M2b.5 is the large
one, like M2a.6 was).

### M2b.1 — Boundary value model: constructors, typed readers, `Kind` — **DONE**

**Landed** (doodle-rust `1020795`). New `machine/boundary.rs`:
`make_int/bool/nil/float/string/bytes/list` + `list_append`, and readers
`kind_of`/`as_int`/`as_bool`/`as_float`/`is_nil`/`string_bytes`/`as_bytes`/
`list_length`/`list_get`, plus `Kind`/`ValueError`. Determinism honored on the
construct path: `make_string` validates UTF-8 + NFC-normalizes; `make_float`
canonicalizes NaN to `0x7FF8…` while passing ±∞/−0.0 through bit-for-bit
(S-56/S-28). `kind_of` maps `Int`/`BigInt` → `Kind::Int`; `as_int` on a bignum
→ `IntOutOfRange`. Supporting: accounting-aware `Heap::list_push`;
`HandleTable::resolve` promoted from test-only. **5-lens read-only review:**
0 code defects; 2 folded — a MAJOR *test-coverage* gap (the new `list_push`
byte-accounting had no test; a deleted charge survived the suite → added a
heap test asserting the exact charge and its sweep-reclamation to baseline)
and a MINOR doc gap (`list_get` mints a host-owned handle — release
obligation now documented). 14 unit tests.

- **Goal.** The public host↔value API (E§4.3/§4.4) over the demo-subset
  kinds, so foreign functions and observation can construct and read
  values. This is the foundation every later M2b item builds on.
- **Lands.** Promote the `#[cfg(test)]` `resolve` to a public generation-
  checked reader; add `kind_of(Handle) -> Kind` (a public `Kind` enum over
  the demo subset), typed readers `as_int`/`as_bool`/`as_float`/`is_nil`
  and `string_bytes` (NFC UTF-8, per E§4.3 normative), list readers
  (`list_length`/`list_get`); constructors `make_int`/`make_bool`/
  `make_nil`/`make_float` (NaN-canonicalizing, S-28) / `make_string`
  (validate UTF-8 + NFC) / `make_bytes` / `make_list` + `list_append`. Each
  constructor interns through the handle table so the result is a GC root
  the host owns. Pin S-35 handle-retain wording in E.
- **Design refs.** E§4.2/§4.3/§4.4; MD §16 (handles).
- **Tests.** Round-trip `string_bytes(make_string(b)) == b` for NFC input;
  non-NFC normalizes; invalid UTF-8 rejected; `kind_of` over every demo
  kind; `make_float(NaN)` canonical; a constructed value survives a forced
  collect (it is handle-rooted); stale-handle errors.
- **Review.** Determinism (NFC/NaN canonicalization on the construct path),
  handle-rooting completeness, generation-check coverage.
- **Depends on.** M2a.11 (`HandleTable`) — done.

### M2b.2 — Foreign-function registry + synchronous foreign functions + `print` (S-43) — **DONE**

**Landed** (doodle-rust `f0d4f1a`). New `machine/intrinsic.rs`: `Registry`
(register-before-load; duplicate-name and built-in-type-value collision →
`HostError`; registration order is replay identity), the `Intrinsic`
descriptor (kind, params, `fn`-pointer callback), `IntrinsicCtx` (bound args
+ heap + output sink), inline synchronous `apply`, and the `print` intrinsic
+ a **provisional** value renderer (Stringable stand-in → M4/M9a). `CalObj`
gained a `CallableTarget::{Source,Intrinsic}` split (`call.rs` `apply`
branches to run an intrinsic inline — never a frame; frame/return sites read
`source_id()`); object defs moved to `heap/objects.rs` (length). Namespace
seeding `globals → BUILTINS → intrinsics` (user shadows intrinsic); an output
sink + `Instance::{load_with_intrinsics, output}`. Conformance runner
registers `print`, matches `expect-out`; the last SKIP is gone (**66/0/0**).
Foreign defaults are inline-only (heap-backed → S-42/M7). **6-lens read-only
review:** 0 code-safety defects (representation-safety/determinism/binding/GC
lenses clean); 2 folded — a MINOR call-parity gap (a `do…end` passed to a
block-less intrinsic silently dropped → now raises like a source callee) and
a MINOR conformance-harness gap (a run fixture with no `expect-out` didn't
check output → spurious output now FAILs). 9 intrinsic tests.

**Discovered + wired:** the machine never evaluated **string literals**
(`StrLit`) — a real M2a demo-subset gap; M2b.2 wires non-interpolated string
literals (alloc the NFC string). **Interpolation stays M4** (Stringable).

- **Goal.** Register host callbacks and call them from Doodle; ship `print`
  as the first native intrinsic, unblocking the `expect-out` conformance
  run-tests.
- **Lands.** A **foreign-function descriptor** (kind `to`/`fn`; ordinary
  params with defaults; at most one trailing block param — S-42-lite) and a
  **registry** on the instance. **S-43 provisional registration:** global-
  scope foreign bindings seeded into the namespace at load (after module
  globals + the builtin type prelude, so a user global still wins the linear
  scan — mirroring `types::BUILTINS`), each a `Value::Callable` tagged
  foreign. The apply path (`call.rs`) branches on a foreign callee:
  pre-bind positional/keyword/default arguments per L§8.3 (reuse
  `bind_arguments`), invoke the callback with an **activation** exposing the
  bound args as handles + the instance + host data; the callback returns a
  value (`fn`) / nothing (`to`) / a raise. `print` reads its string argument
  (via `string_bytes`) and appends to a host-owned **output sink** (captured
  deterministically — no direct stdout on the Doodle-observable path). A
  foreign `to` used in expression position hits the runtime Void backstop
  (`take_value` raises) — the static voidcheck only knows current-module
  `to`s. Spec: E§5.1/§5.2/§5.5 S-43 mechanism + its M5 retirement note; S-19
  sync-FF determinism note.
- **Design refs.** E§5.1/§5.2; MD §14 (synchronous foreign callback; args
  rooted for the callback's duration); MD §8 (callable identity — foreign
  callables get a distinct `CalObj`/tag).
- **Tests.** `print("hi")` appends to the sink; `expect-out` fixtures
  (`arith-001`, …) now PASS in the conformance runner (flip the SKIP);
  keyword + default binding into a foreign call; foreign `to` in expression
  position raises; a foreign callback that raises surfaces as `Raised`;
  determinism gate over a corpus that prints.
- **Review.** Argument-binding parity with Doodle calls (L§8.3), the
  namespace-seeding order (user-global-wins), foreign-`to` Void discipline,
  output-sink determinism.
- **Depends on.** M2b.1.

### M2b.3 — The drive-state machine: resume, directives, terminal guards — **DONE**

**Landed** (doodle-rust `934481b`). `drive::run` is now a resumable loop
(`Ready`/`Paused` → step → `Completed`/`Raised`/`Faulted`/`Paused`), plus a
`resolve(Resolution)` entry (phase-guarded; the resume path is M2b.4).
**Basic `Step*`** (user ruling): `step()` reports statement-level safe-point
crossings + their frame depth; `should_pause` implements `Step`/`StepInto`
(next safe point), `StepOver` (`depth ≤ anchor`), `StepOut` (`depth < anchor`);
`RunToCompletion`/`Continue` never pause (breakpoints/raise-trap → M6). **E§3.3
outcome↔state correspondence** with the new terminal **`InstanceState::Raised`**
(user-ratified; distinct from `Faulted`, `is_terminal()`). Host-contract phase
guards (re-driving a terminal / `resolve`-ing a non-`Suspended` instance →
debug-assert + `Faulted(Internal)`). **5-lens review: 1 MAJOR folded** — a
root-caused `StepOut` bug: a frame-popping unwind (`return`/block `break`)
reported no safe point at the return depth, so `StepOut` overshot into sibling
calls; now the **settling** unwind transition reports the return safe point and
runs `limits::safe_point` like the fall-through `ReturnBarrier` (behavior
change: return-via-unwind now ticks a return safe point too, consistent with
E§7.4; determinism gate holds). 8 drive-directive tests + a StepOut regression.

- **Goal.** Replace the single-shot `run()` with a real resumable loop over
  `InstanceState`, honoring directives and rejecting mis-phased drives — the
  spine capabilities and reentrancy need.
- **Lands.** `run(instance, directive)` (start, or continue after `Paused`)
  and `resolve(instance, resolution)` (continue after `Suspended` — the
  latter exercised at M2b.4). Directive semantics at the **basic**
  granularity (per open-decision #1): `RunToCompletion` fast; `Continue`
  runs to the next capability/fault/completion; `Step`/`StepInto`/
  `StepOver`/`StepOut` pause at the next matching statement-level safe point
  by frame depth → `Paused(Step)`. **Terminal-state guards:** re-driving a
  `Completed`/`Faulted` instance, or `resolve`-ing one with no pending
  suspension, is a host-contract fault (debug-asserted + a returned fault),
  not UB — replacing the M2a.3a `debug_assert!(Ready)` single-drive
  stopgap. **Pin the post-`Raised`/`Faulted` state in E§3.3** (the flagged
  delta). Split `drive.rs` if it crosses the length limit.
- **Design refs.** E§3.3, §7.1/§7.2/§7.3/§7.4; MD §17 (position at a return
  safe point).
- **Tests.** `Step` pauses statement-by-statement over a multi-statement
  program, resuming to `Completed`; `StepOver` treats a call as one step;
  `StepOut` runs to the caller's next safe point; a `Continue` with no
  breakpoints behaves as `RunToCompletion`; re-driving a completed instance
  faults; drive-directive determinism (same terminal under every directive
  mix — exit criterion 5, first slice).
- **Review.** Directive-depth correctness against frame identity, terminal-
  guard completeness, that `Step` observes Void-between-statements
  correctly (M2a.6 register clearing), determinism across directive mixes.
- **Depends on.** M2b.2 (something callable to step through) — soft; could
  precede it, but stepping over a `print` is the natural test.

### M2b.4 — Suspending capabilities + `resolve` + capability requests — **DONE**

**Landed** (doodle-rust `6905717`). `Intrinsic` gained a
`ForeignBody::{Sync, Capability}`; a capability call parks a `PendingRequest`
(capability id = registry index, args) and the drive loop returns
`Suspended(CapabilityRequest{capability, args as host-owned handles})` — **no
state torn down** (MD §14). `resolve(Value)` injects the result (or Void for a
`to` capability) and resumes the drive under the in-force directive;
`resolve(Raise)` surfaces `Raised` at the call site — **provisional
(user-ruled): a `HostRaised` kind + rendered message**, the value-carrying
exception `rescue` binds is M4 (E§9; filed in the spec-delta queue). Parked
args are GC roots while `Suspended`; capability id is stable (S-43 order) for
replay (E§11). Scripted `read_line` capability. `Value`/index types split to
`machine/value.rs` (file length). **6-lens review: 1 MAJOR folded** — a
`resolve` on a stale handle returned `Faulted` but left the instance
`Suspended` (resumable half-state, violating E§3.3 outcome↔state); now the
suspension is cleared and the instance is left terminally `Faulted`. 12
drive-directive tests (suspend/resolve-value, resolve-raise, request identity,
stale-handle fault, replay determinism).

- **Goal.** A capability call yields to the host and resumes with the
  host's value — exit criterion 1 (`read_line`).
- **Lands.** A **capability descriptor** (a stable capability identity +
  kind; no inline callback). Calling a capability reaches `apply` and,
  instead of running a callback, stores `CapabilityRequest { capability_id,
  args }` in the machine's `pending` and returns `Suspended(request)` — **no
  state torn down** (MD §14). `resolve(Value(h))` sets `reg` to the handle's
  value and continues under the directive in force; `resolve(Raise(h))`
  enters the §12 raise path at the call site (trace captured pre-unwind). A
  `to` capability resolves with `nil`/no-value. Capability identity is
  **stable across runs** (E§7.5/§11 — an index/name, never address-derived).
  A scripted `read_line` test capability. Confirm S-17 (observation while
  Suspended).
- **Design refs.** E§5.3/§7.5; MD §14 (`pending`, park-and-resume,
  sufficiency: partial results live in conts, not `reg`).
- **Tests.** `read_line` suspends with the right request identity + arg
  handles; `resolve(Value)` delivers the result as the call's value;
  `resolve(Raise)` raises at the call site; the same script replays to a
  byte-identical trace (determinism/E§11); observation (position/stack)
  works while Suspended.
- **Review.** Park sufficiency (no live partial result in `reg`), request-
  identity stability/determinism, the raise-on-resolve trace-capture point.
- **Depends on.** M2b.3.

### M2b.5 — Reentrant drives + native block-consuming functions + S-46

**Split for size (M2a.6-scale).** **M2b.5a** (reentrant drive + native
`each`, complete/`continue`/raise) and **M2b.5b** (S-46 non-local exits) are
both **DONE**. The split sequenced the large item — it did not defer scope.

#### M2b.5a — reentrant drives + native `each` — **DONE**

**Landed** (doodle-rust `b02b5f8`). `IntrinsicCtx` became a rich mutable
step-context with `invoke_block(args)`, which runs a **nested drive** on the
shared heap stack (push the block frame at a `Consumer::Native` boundary, step
to `frames.len() ≤ boundary`, then `Completed` / a nested raise propagates /
`continue` = normal completion). A limit inside the nested drive can't flow
through the Raise-typed `apply` chain, so it parks `Machine.reentry_fault`,
which `step` surfaces as `Fault`. Native **`each`** (a `to` over a list + block)
iterates a fixed count over the live heap list. `break`/`return` crossing the
native boundary raise `Unsupported` (S-46 → 5b). Also wired **list-literal
evaluation** (`Node::List`, a demo-subset gap like `StrLit`). `namespace`
threaded through the call path so a reentrant callback can `step`. **5-lens
review: 2 CRITICAL/MAJOR folded** — (1) a program recursing *through* a native
consumer overflowed the host **Rust stack** (SIGABRT) rather than faulting; now
`MAX_REENTRY_DEPTH` (the MD §14 drive-depth I'd dropped) faults it with
`StackDepth`. (2) the GC-rooting fix (`foreign_roots` roots in-flight foreign
args, MD §15 — I'd caught a use-after-free of heap-valued `each` elements) had
an *ineffective* test (collected between top-level steps, missing the nested
drive); now a `gc_every_safe_point` knob collects inside it. Files split
(intrinsic→dir, `lifecycle.rs`) for length. 10 `each`/reentrancy tests.

*Provisional filed:* `MAX_REENTRY_DEPTH = 64` is a flat Rust-stack-safety bound;
a stack-size-aware / host-configured reentrancy limit is future work (M3/M7).
`Consumer::Native` is a unit variant at 5a; **5b re-adds the boundary depth**
(MD §14) for the `NonLocalExit` resume.

#### M2b.5b — S-46 non-local exits — **DONE**

**Landed** (doodle-rust `bfc9e6f`; spec E§7.6/§5.4 + App B.1 + App C S-46
RESOLVED in the same discussions push). A `break`/`return` inside a block
invoked by a native consumer (`each`) now crosses the host boundary as the
MD §12 **`NonLocalExit`** mechanism instead of the M2b.5a `Unsupported`
stub — a native consumer is program-observably identical to a
Doodle-written block-consumer for every exit kind (S-46 parity).

- **Mechanism.** `Consumer::Native { boundary }` carries the native call's
  frame depth. An exit crossing it leaves the `Unwind` **parked** at the
  boundary (`Unwind::NativeBreak`, or a `return`'s existing punch-through);
  the reentrant nested drive detects it and returns
  `BlockResult::NonLocalExit`; the callback returns promptly; and
  `intrinsic::apply` resumes the parked exit at the foreign call's apply
  site via `unwind::resume_native_boundary` — a `break` targeting *that*
  call completes it with the value, a `return`/outer break stays parked and
  unwinds past it in the enclosing drive (nested consumers compose
  innermost-out). `continue` stays a normal completion (never a
  `NonLocalExit`). A valued `break` to the procedure `each` raises
  `NoValueDestination`, matching the Doodle `to`-consumer path (open S-10).
- **Host-contract faults (S-46 rider 3 / S-16 family).** After a
  `NonLocalExit` the callback must return promptly with no result; driving
  the block again (`invoke_block` with an unwind already parked) or
  returning a value / a raise **faults** (`Faulted(Internal)`) rather than
  re-running the block or stomping the parked exit's target. Engine-authored
  intrinsics comply; this backstops a misbehaving host callback (C-ABI, M7).
- **Hygiene.** Split the demo intrinsics + renderer into
  `intrinsic/builtins.rs` so `mod.rs` stays within the length limit.
- **Tests.** break-ends-`each`, return-crosses-`each`, nested-`each` break
  targets the inner consumer, nested-`each` return unwinds through both,
  valued-break-to-procedure raises, step/run determinism over the break
  path, and two host-contract faults (re-drive / value after a
  `NonLocalExit`, via a test-only misbehaving intrinsic).
- **Review.** 5-lens read-only adversarial workflow (find→verify): **0
  confirmed defects**; the one surfaced spec-parity residual — the E§7.6
  host-contract fault was documented but not enforced — was folded (the
  fault enforcement above).

### M2b.6 — Foreign values + finalizers — **DONE**

**Landed** (doodle-rust `e7203cb`; spec E§4.5 finalizer-timing wording
tightened in the same discussions push). `Value::Foreign` is now
allocatable: a `foreigns` slab of `ForeignObj { tag, ptr, finalizer }`,
constructed at the boundary via `make_foreign(tag, ptr, finalizer?)` and read
with `foreign_tag`/`foreign_ptr`. A foreign value is **inert** to Doodle —
identity-typed (L§4.13), no fields, arithmetic on it raises — reachable
through the GC like any leaf.

- **Finalizers (E§4.5), host-side only, exactly once.** At **GC** for a dead
  value: the index-order sweep **takes** the finalizer out of the dying object
  (so it can never run twice) and queues it; the queue runs **after** the
  collection completes (MD §15). At **`destroy`** for every still-live foreign:
  run once — implemented in `Drop for Instance`, so a plain drop finalizes too
  (no leaked resource); a `destroy(self)` names the E§3.1 op. Exactly-once
  holds across the two (a GC-finalized value's finalizer is already taken, so
  destroy never re-runs it). Each finalizer call is isolated with
  `catch_unwind`: a finalizer **must not unwind** (the M7 C-ABI form cannot),
  but a buggy one that does is contained so it neither skips its peers'
  finalizers (leaking their resources) nor aborts the host during `Drop`.
- **Determinism (E§11).** The finalizer is host state: uncounted in
  `bytes_allocated` (fixed `foreign_payload`), so it can't shift GC triggering;
  not snapshotted (replay re-supplies it); unable to re-enter the instance — so
  its effects never perturb the deterministic run.
- **Spec.** Tightened E§4.5 to state exactly-once, run-after-collection /
  at-destroy-for-live, non-observability, and the must-not-unwind contract.
- **Tests.** tag/ptr/kind round-trip; wrong-kind reader; finalizer once at GC
  (with the ptr) and not re-run; a retained foreign not finalized; destroy
  finalizes every live foreign once; no double-finalize across GC+destroy; a
  no-finalizer foreign inert; a panicking finalizer isolated from its peers;
  and a foreign injected into a running program is inert (arithmetic raises)
  yet still finalizes once at destroy.
- **Review.** 5-lens read-only adversarial workflow (find→verify): **1
  confirmed** (a panicking finalizer leaking its peers / aborting in `Drop`),
  folded via `catch_unwind` isolation + a test.

*Provisional filed (S-42-lite):* the in-engine finalizer shape is a boxed
`FnOnce(u64)` receiving the host ptr; the C-ABI form (`extern "C" fn(void*)`)
is deferred to the full S-42 host-callback FFI (M7).

### M2b.7 — Cancellation + observation (position/stack) + M2b exit review — **DONE**

**Landed** (doodle-rust `c181f7e`). **Cancellation** (E§10.1): a cloneable,
thread-safe `CancelToken` sets a flag the engine polls at each safe point;
once set the drive arms `Unwind::Cancel`, which tears the stack down **one
frame per transition** (the MD §12 unwind path — running each frame's block/
`with` cleanup at M4; inert now, no cleanup conts yet) and faults
`Faulted(Cancelled)` at the empty-stack terminus — terminal, non-resumable,
**not catchable** by Doodle. Works at the top level, across a suspend/resume,
and *through a native block-consumer's reentrant drive* (the parked `Cancel`
crosses the S-46 boundary: `resume_native_boundary` declines it, the
enclosing drive tears down). **Observation** (E§8.1/§8.2, the M2b minimum,
new `machine/observe.rs`): `current_position()` → module id + byte `Span`
(the host renders line/column — the engine exposes positions, not text; an
end-of-body drain reports an end-of-module position, never `(0,0)`);
`stack_walk()` → innermost-first frames, each a callable handle + call-site
span + tail-iteration count (frames gained a `call_site`). The **step
budget** was already wired to the public surface (Config.limits →
FusedCounter, M2a.9/M2a.11) — confirmed, no change.

**M2b exit review (5-lens read-only find→verify, + a re-run determinism
lens): exit-criteria and determinism CLEAN; 3 minor/nit findings folded** —
(1) a vacuous reentrant-cancel test replaced with one that cancels from
*inside* the block (via a test-only `IntrinsicCtx::request_cancel`),
actually exercising the S-46 cancel-across-the-boundary teardown; (2)
`poll_cancel` no longer arms `Cancel` on an empty stack (a cancel racing
exactly with completion loses to it — no dangling unwind); (3) the
`current_position` end-of-body fallback above.

*Cancel-vs-completion race RESOLVED (user, 2026-08-03; E§10.1 pinned +
generalized):* cancellation is a request about *future* work — once observed,
no further program work runs, and the outcome reports what actually happened.
It takes effect only at a safe point where program work remains; otherwise the
run's own terminal outcome stands (a cancel first observed at completion →
`Completed`, at an uncaught raise reaching the boundary → `Raised`), and cancel
on an already-terminal instance is an idempotent no-op. The implementation
already satisfies the generalized rule (a raise/limit short-circuits the cancel
poll; a terminal instance is never re-polled).

**With M2b.7, milestone M2b (the host/embedding layer) is complete:**
M2b.1–M2b.7 all landed and reviewed; the drive-state machine, boundary value
model, intrinsic foreign functions + suspending capabilities, reentrant
drives + native block-consumers + S-46, foreign values + finalizers,
cancellation, and the minimal observation surface all ship. The richer
E§8.2 frame surface (named locals, dynamic-parameter bindings) and the
debugger facilities (breakpoints, raise-trap, per-subexpression stepping)
are **M4/M6** as planned.

---

## Notes on ordering and risk

- **Critical path:** M2b.1 → M2b.2 (unblocks `expect-out`) → M2b.3 →
  M2b.4 → M2b.5 (the risk peak) → M2b.6/M2b.7. M2b.6 (foreign values) is
  largely independent of M2b.4/M2b.5 and could interleave.
- **Risk peak:** M2b.5 (reentrant drives + S-46), as M2a.6 (blocks/unwind)
  was in M2a — the park-and-resume-across-`HostBoundary` path is the
  subtle one. It gets the widest review.
- **Determinism** is exercised continuously: the M2a.12 GC-stress harness
  extends to cover printed output, capability resolutions (replay), and
  directive mixes (E§7.7) — the boundary must not leak order (E§11).
- **Spec-delta discipline:** the starred deltas (S-43, S-46) need a fresh
  ask when reached; the rest land the pre-agreed direction with the item.
