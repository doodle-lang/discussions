# Working Plan: Milestone M2a — Machine Core

**Status:** Proposed (awaiting ratification). The *design* it implements —
`plan/machine-design.md` v0.2.1 — is **already accepted** (M2a gate
satisfied, 2026-07-10); this file only **sequences** that design into
session-sized work items. · **Parent:** `implementation.md` §5 M2a `[L]`
(the largest milestone; widest error bars). · Conventions as in
`plan-m1.md`/`plan-m0.md`.

Decomposition of M2a (the CESK machine, slab heap + GC v0, handles, over
the demo subset) into ordered, session-sized work items. Each item lands
its doodle-core tests green and hygiene clean, and gets a minimal
read-only adversarial review before it lands (the M1 rhythm).

**Design authority.** `machine-design.md` is the source of truth for the
*mechanisms* (value repr §3, heap/slabs §4, frames/`Cont` §8, unwinding
§12, PTC §11, GC §15, handles §16). Rust shapes there are sketches;
renaming/reshuffling is fine, changing a mechanism requires revising that
doc first (its intro). Where an item below says "per MD §N", that section
governs.

**Ratification note.** Once ratified, the *sequencing* and the
*already-accepted* spec resolutions cited on items (S-9, S-55, E§7.2
top-level completion — all pinned by the accepted machine-design) are
pre-approved: landing the spec edit that implements exactly the stated
resolution needs no further sign-off. The **open** rulings flagged below
(S-10, S-12's huge-exponent half, S-41) each need a fresh ask when reached
(CLAUDE.md: never change semantics without asking).

---

## The demo subset (what M2a runs)

From `implementation.md` §5 M2a. **In:** numbers (int + bignum promotion,
float), bools, nil, minimal strings, bytes, lists; built-in type values +
`is`; `let`/`const`/assignment; `if`/`while`/`loop`; operators with strict
booleans and Void enforcement; `to`/`fn`/anonymous `fn` with keyword args
and defaults; blocks with `continue`/`break`/`return` incl. punch-through;
PTC (ring buffer + tail counters); per-statement safe points + fused fuel;
heap + non-tail stack-depth limits.

**Out (later milestones), even though the machine's *slots* for them exist
now:** dicts + value/ref records + structural `==` (M4); `with`/`parameter`
dynamic binding (M4); `try`/`rescue`/`raise` surface syntax (M4 — but the
`Unwind` mechanism and `WithRestore`/`TryHandler` cont *categories* are
built at M2a, §12); modules/imports/protocols/dispatch (M5);
capabilities/foreign functions/`print` (M2b). A runtime **type error**
(e.g. `1 + true`, non-bool `if` condition) still surfaces as `Raised` at
M2a even though user `raise` syntax is M4 — the machine can raise
engine-generated exceptions.

## M2a exit criteria (from the plan)

1. A 10⁷-iteration tail-recursive loop runs in **constant memory** with the
   tail counter reading 10⁷.
2. Deep **non-tail** recursion trips `Faulted(LimitExceeded(stack))`.
3. An unbounded allocation loop trips `Faulted(LimitExceeded(heap))` **at a
   deterministic step**.
4. **GC-stress determinism gate green** over the M2a corpus (run twice + GC
   every safe point, diff traces).
5. Closures capture **loop-fresh** bindings correctly.

Observed via doodle-core Rust integration tests + the determinism harness,
**not** stdout: `expect-out` needs `print` (M2b), so at M2a only raise-only
conformance run-tests execute (conformance/README.md).

## Spec-delta obligations coming due in M2a

Resolve each **in the spec** as part of the item that implements it (plan
§8: edit + decision-log entry + conformance/test). Flagged here so none is
silently deferred (CLAUDE.md).

- **E§7.2 — top-level `Completed` value.** The M0 skeleton provisionally
  returns the result register; real top-level completion is expected to be
  **Void (`None`)**. Pin in E§7.2 with **M2a.2** (when the real drive
  replaces the placeholder). *(Already-agreed direction; landing the edit.)*
- **S-9 (L§7.10) — `break`/`continue` punch through `with`.** MD §12's
  punch-through-with-restoration **is** the proposed resolution; land the
  L§7.10 edit **by M2a** (with **M2a.6**, the unwinder — even though `with`
  itself is M4, the mechanism and its spec statement are fixed now).
  *(Already-agreed direction via the accepted MD §12.)*
- **S-55 follow-up — reuse kind-check + mixed-kind parity tests.** Land with
  **M2a.7** (PTC): the apply-time kind gate + conformance tests (to-tail-fn
  value discarded; fn-tail-to falls off at the fn's own completion).
  *(Spec already landed; this is the code + tests.)*
- **S-41 (E§3.1) — `create(config)` target-Unicode-version vs the build-time
  pin.** Engine reports its pinned version; `create` fails an unsupported
  request, or the field is dropped. Resolve by M2a (with **M2a.11/handles**
  or the instance-config item). **Needs a ruling** (two candidate shapes).

**Surfacing during M2a, may need a ruling when reached (flag, don't
guess):** S-10 (break-with-value where the consumer has no value slot —
discard vs error; **M2a.6**), S-12's remaining half (huge-exponent
resource behavior of bignum `Int ** Int`; **M2a.3**).
**S-28 is RESOLVED (user, 2026-07-28; App C + spec landed):**
exact-value `==` across kinds, `-0.0 == 0.0 == 0`, one canonical
self-equal NaN (E§4.3 canonicalization), NaN ordering raises,
first-inserted key retained, hash coherence — the comparison half is
*implemented per the ruling* at **M2a.3**; the dict-key **hash** half
at M4. **S-56 is RESOLVED (user, 2026-07-28; App C + spec landed):
raise on any nonfinite float-arithmetic result** (underflow fine;
result-based; int→float widening included; overflowing literal =
front-end static error) — closes S-12's `0 ** negative` corner by
subsumption; **M2a.3** implements the finiteness check.

---

## Work items

Dependency-ordered; each is one session-ish (a couple may split). The
`machine.rs`/`drive.rs` M0 placeholders are **replaced**, not extended —
they walk a bare `Ast`; the real machine consumes a `ResolvedModule`.

### M2a.1 — Heap foundation: slabs, objects, allocation accounting (no GC) — **DONE**
*(doodle-rust: generic `Slab<T>` + `Heap` {strings, bytes, lists} + len-based
deterministic payload accounting + serial identity + `same_ref`; read-only
review clean, MAJOR folded by dropping the untracked-growth accessor. Follow-up
tracked: the payload-count accounting-model gap needs an MD §4/§15 ruling before
M2a.9 — see claude-todo.)*

`Heap` with one `Slab<T>` per demo-subset kind (MD §4): `bigints`,
`strings`, `bytes`, `lists`, `callables`, `cells`, `types` (dicts/records
join at M4). `Slot = Occupied { obj, mark, serial } | Free { next_free }`;
`free_head`; `live_count`; `bytes_allocated` (program-driven payload bytes,
**excludes** the §5 grapheme memo); `alloc_serial` (monotonic, stamped on
every alloc). Identity = slab index; `serial` the identity-derived scalar;
`is_same` = same variant + index (MD §4). **No mark/sweep yet** —
allocate-only; the free list exists but is only exercised once GC lands
(M2a.10). *Accept:* alloc → distinct indices + monotonic serials;
`bytes_allocated` tracks payloads; `is_same` correct; unit tests.

### M2a.2 — Machine skeleton: `Instance`/`Machine` over `ResolvedModule`; frames, `Cont`, `step()`; literals — **DONE**
*(doodle-rust: `Machine`/`Frame`/`FrameKind`/`Cont`/`step` in `machine/{frame,cont,step}.rs`;
the M0 `Instance`/`drive` walk replaced with load(`ResolvedModule`) → `ModuleTopLevel`
frame → `Seq`/`Eval` conts; immediate + bytes literals; module-top-level Void return.
**E§7.2 resolved** — top-level completes `Completed(None)` (prose + Appendix B.1 entry;
`drive_smoke` updated). `mode: run` conformance still SKIPs until the run-arm at M2a.12.
**Deferred from this item (review-flagged, tracked to M2a.4):** `load` does not yet
create module cells for globals — harmless now (a global-declaring statement isn't an
`ExprStmt`, so it hits the not-yet-implemented path before a cell is read), but M2a.4
must add it. **Forward note (M2a.9):** `reg` is not reset at a statement boundary, so a
future safe-point observer would read the prior statement's value — pin the
statement-boundary register semantics when safe points land.)*

Replace the M0 `machine.rs` `Instance` and `drive.rs` walk. `Machine` (MD
§8): `frames`, `reg: Option<Value>`, `ring`, `fuel` (stub until M2a.9).
`Frame` (kind/`locals`/`conts`/`serial`/…). The `Cont` enum seeded with
`Seq { body, next }` (statement boundary = safe point). `step()` pops the
top cont of the top frame (returns from the frame when empty). Load a
`ResolvedModule`: create module cells for `globals` (MD §6; `GlobalKind` →
`CellKind`), push a `ModuleTopLevel` frame, `Seq` the top-level statements.
Evaluate literal expression statements into `reg`. **Resolve E§7.2**:
top-level drive completes `Completed(None)` (Void). *Accept:* a literal /
`ExprStmt` program drives to `Completed(None)` through the real machine;
lifecycle states (E§3.3) transition; integration test.

### M2a.3 — Expressions: operators, numeric tower, strict booleans, Void enforcement
**Split into 3a (arithmetic + raise) and 3b (comparison + boolean).**

**M2a.3a — arithmetic + numeric tower + the raise path — DONE.** *(doodle-rust:
`machine/arith.rs` — `+ - * / // % **` + unary `- +` over the exact integer tower
(bignum promotion + demote-on-fit, canonical-int invariant) and floats (S-56
finite-result rule on every float path, incl. the int→float widening; div/mod by
zero raises; floored `//`/`%` via `num_integer` for ints and `fmod`-adjust for
floats). The **raise path**: `machine/error.rs` (`Exception`/`ExceptionKind`/
`Trace`/`Raise`); `step` returns `Result`, a failing op propagates uncaught →
`Outcome::Raised` (handlers/§12 unwind are M4/M2a.6). Void enforcement via
`take_value`. Read-only review clean bar minor folds: unary-`+` finiteness gate,
robust float `%`, coverage tests, a re-drive guard. S-12's `0 ** negative` closed
by S-56 subsumption; the huge-exponent resource half is provisional
(`ExponentTooLarge`) until the M2a.9 limits. The deferred S-56 overflowing-float-
**literal** diagnostic (L§3.6.2) has an `#[ignore]`d tripwire test; discovered
delta: post-`Raised` instance state — provisionally `Faulted`, pin in E§3.3.)*

**M2a.3b — comparison + equality (S-28) + strict booleans — DONE.** *(doodle-rust:
`machine/compare.rs` — `< > <= >=`, `== !=`, and boolean `not`; `and`/`or`
short-circuit via new `AndRhs`/`OrRhs`/`AssertBool` conts in `step`. Equality is
total/reflexive/never-raising; numbers compare by **exact** value across kinds —
the int↔float comparison decomposes the `f64` into `mantissa·2^exp` and scales the
integer side in `BigInt`, so it is exact beyond 2⁵³ (S-28). `-0.0 == 0.0 == 0`;
one self-equal canonical NaN; NaN ordering raises; ordering defined for numbers
and strings only, else raises. Verified three ways: an independent 1.2M-pair
cross-check of the exact int↔float comparison vs exact rationals (0 mismatches);
a 6-lens adversarial verification workflow (0 findings); 60 lib + 9 integration
tests. NaN/`-0.0` aren't source-constructible at M2a.3, so those paths are
helper-tested.)*

Expression-plumbing conts (`BinRhs`/`BinApply`/`UnaryApply`, MD §8). Numeric
tower: `i64` fast path with **overflow → bignum promotion** and
**demote-on-fit** (the canonical-int invariant, MD §3); `/` → float; `//`,
`%`, `**` (S-12's huge-exponent resource half — **ruling**; `0 **
negative` raises per resolved S-56); the S-56 finite-result rule (any
float op whose IEEE result is nonfinite raises — underflow fine,
int→float widening included); comparisons per resolved S-28 (exact
cross-kind; `-0.0 == 0.0`; single canonical self-equal NaN,
canonicalized at every NaN-producing op; NaN ordering raises). Strict
booleans: `and`/`or`/`not`
and every condition position **raise a runtime type error** on a non-bool
(no truthiness, L§4.1). **Void enforcement** (L§6.11): a cont that uses
`reg` as a value errors if it is `None` (belt-and-suspenders behind the
resolver's static S-6 check — covers the dynamic cases). Runtime errors →
`Raised`. *Accept:* arithmetic incl. a bignum-promoting product; `1 + true`
raises; comparison/`==` per the ruling; raise-only conformance fixtures.

### M2a.4 — Statements & intra-frame control: `let`/`const`/assign, `if`, `while`, `loop`
**Split into 4a (bindings) and 4b (control flow).**

**M2a.4a — module cells + `let`/`const`/assign — DONE.** *(doodle-rust: a `cells`
slab + `CellObj { value: Option<Value> }`; `Instance::load` allocates one
uninitialized cell per module global and builds the name→cell namespace;
`BindLet`/`AssignTo` conts; `read_ref`/`bind_let`/`assign_to` in `machine/
control.rs` dispatch on the resolver's decision — a module decl has no resolution
→ a cell, a nested decl → `LocalSlot`. **Forward-reference/hoisting resolved
(user-approved TDZ model):** cells created at load, `let`/`const` fill them in
execution order, a read before then raises `UsedBeforeDefined`; a name with no
binding raises `NameNotDefined`. Assignability stays static (S-6 rule 2a), so no
runtime const check; `CellKind` deferred until parameters/dispatch need it.
`Frame` gains `locals` (empty at top level with no constructs) for 4b. Verified
by a 3-lens adversarial workflow cross-checking the machine vs. the resolver's
output — 0 findings.)*

**M2a.4b — `if`/`while`/`loop` — DONE.** *(doodle-rust:
`IfChoose`/`WhileCheck`/`LoopReloop` conts carrying the construct `NodeId` (so
M2a.6 exits can target them); strict-`Bool` conditions (non-bool → `TypeMismatch`);
`else if` = the resolver's flattened arms; `if`-expression leaves the branch
value in the register (void-check guarantees producing branches + `else`);
`while`/`loop` yield Void; construct bodies run in the enclosing frame — a `let`
inside a loop is the first real use of `Frame::locals`. Read-only review: no
defects above NIT. **Forward note (M2a.6):** loop-body local slots aren't reset
per iteration — safe today (the resolver binds references in source order and
there is no `break`/`continue`/closure yet, so a `let` always runs before any
read resolving to it), but re-verify when non-linear intra-body control flow
lands.)*

### M2a.5 — Calls: `to`/`fn`/anonymous `fn`, args, `Callable` frames, `is`/type values
`EvalArgs`/`Apply` conts; argument binding per L§8.3 at `Apply` (positional
+ **keyword args** + **defaults**, defaults evaluated in the callee
activation per L§8.2). Push a `Callable` frame with a `ReturnBarrier` cont +
the body `Seq`. Plain module-level `to`/`fn` intern **one canonical
`CalObj`** at load (identity, MD §8); anon `fn` makes its own (captures at
M2a.8). Built-in **type values** (`types` slab) + `is` (type cases,
non-protocol). **Non-tail only here** (reuse is M2a.7). *Accept:* a
non-recursive `fn`/`to` call tree; keyword+default binding; `x is Int`;
`to` completes Void.

### M2a.6 — Blocks + three-tier exits + the unwind mechanism (§12)
The hardest item. `Block` frames (`defining`/`defining_serial`/`consumer`,
MD §8); block descriptors in `block_param`; **static links** (`BlockOuter`
→ chase the defining chain `hops` times). The single `Unwind` record
(Return/Break/Continue/Cancel; Raise's dynamic-handler search too, for
engine raises). The unwinder pops conts/frames, executing **only**
`WithRestore` (a no-op path until `with` at M4) and — for Raise —
`TryHandler` (inert until `try` at M4). Targets are **resolver-annotated**
(`ExitTarget`): `return`→`HomeCallable` (chase defining chain to the home
callable — *not* the block's consumer), `break`/`continue`→`ThisLoop(node)`
/ `ThisBlock` / `ConsumerCall` (consult the `Block` frame's `Consumer`).
**Punch-through** through intervening block/consumer frames. **Land S-9**
(L§7.10 edit). **S-10 ruling** (break-with-value into a slot-less consumer).
*Accept:* `each`-style block over a list with `break`/`continue`/`return`
reaching the correct target across nesting; `return` from inside a block
exits the *writing* function, not `each`.

### M2a.7 — PTC: tail-call frame reuse + ring buffer + tail counters (S-55)
Executing a marked tail call (MD §11): evict the current occupant into
`ring` (S-34 fields), **completely overwrite** the frame
(`kind`/`call_site`/`locals`/`block_param`/`conts`), **preserve** `serial`
+ logical depth, `tail_count += 1`. The **apply-time kind gate** (S-55): reuse
iff callee kind matches the frame's original callable kind (Block frames
reuse for either); a mismatch pushes an **ordinary** frame (exact non-tail
parity). `debug_assert!(dyn_stack.len() == frame.dyn_depth)` at reuse.
**Land the S-55 follow-up tests** (mixed-kind parity). *Accept:* **exit
criterion 1** (10⁷ tail loop, constant memory, counter = 10⁷); to-tail-fn
discards the value; fn-tail-to falls off at the fn's own completion.

### M2a.8 — Closures: capture cells, loop-fresh bindings
`CalObj` holds the capture cell list; at closure **creation** read each
`CaptureSource` from the enclosing environment (static link from the
creating frame) into the closure's capture slots; at **invocation** splice
those cells into the new frame (MD §7/§10, representation B). Cell-boxed
slots deref through the `CellObj`. Per-iteration fresh scopes (L§5.4) →
loop-created closures capture **distinct** bindings. *Accept:* **exit
criterion 5** (loop-fresh capture); `make_counter` mutates shared state
across calls (S-11); two closures sharing one binding see each other's
writes.

### M2a.9 — Safe points + fused counter + limits
`FusedCounter` (MD §9) = min(slice fuel, step budget, distance-to-next-
armed-event); one decrement-and-branch at each **statement-level** safe
point (`Seq` boundary, call entry, return). Limits: non-tail **stack
depth**, **heap** (`bytes_allocated` threshold), **step budget** →
`Faulted(LimitExceeded(kind))`. GC-trigger + heap-limit checks fire **only**
at statement-level safe points (observation-mode-independent, E§7.7).
**Pin the statement-boundary register semantics (carried from M2a.2):** `reg`
currently retains the prior statement's value between statements, so a safe-point
observer would read it — decide whether a `Seq` boundary clears `reg` to Void
(likely) and implement it with the safe point.
*Accept:* **exit criteria 2 + 3** (deep non-tail recursion → stack fault;
unbounded alloc → heap fault at a deterministic step).

### M2a.10 — GC v0: precise non-moving mark-sweep, index-order
Trigger only at statement safe points, only by `bytes_allocated` crossing a
threshold (MD §15). Roots: handles (M2a.11), every frame's `locals`/
`block_param`/`conts`, `reg`, `unwind`, `dyn_stack`, module namespaces
(incl. `Loading`/`Failed`), `pending` args, `ring` callable refs, native
regs, in-flight sync-call args. **Mark** = explicit work-stack (no recursion
— cycles legal). **Sweep** each slab in **index order**, rebuilding free
lists in index order (determinism). *Accept:* the alloc loop reclaims;
before/after a forced GC, results identical; the M2a corpus unchanged with
GC on.

### M2a.11 — Handles + instance config (S-41)
`HandleTable` (MD §16): generational `u64` handles, `retain`/`release`,
GC roots, generation check **at the host boundary only**. Instance
create/config surface (E§3.1) — **resolve S-41** (Unicode-version field vs
the build-time pin). *Accept:* a retained handle survives GC; a stale
generation is caught at the boundary; `create` behaves per the S-41 ruling.

### M2a.12 — Determinism harness + conformance run-arm + stage gate → Run + M2a exit review
The **GC-stress determinism harness** (a conformance-runner flag / a
doodle-core test mode): run twice, once with GC forced at every safe point,
diff the observable trace. Wire the conformance **`run` executor arm**
(match `expect-raise`; a run test carrying `expect-out` needs `print`, so
it **stays SKIP until M2b** — gate on the test's expectations, not just the
stage scalar). Bump `stage::implemented_through()` → `Some(Stage::Run)`
**atomically** with that arm. Adversarial **M2a exit review**; walk all
five exit criteria. *Accept:* **exit criterion 4** (determinism gate green);
raise-only run fixtures pass; the gate reports Run; review clean.

---

## Notes

- **Conformance staging.** Items land raise-only `#! mode: run` fixtures as
  they gain the behavior; `expect-out` fixtures (e.g. the existing
  `L6.5/arith-001`) stay SKIP until M2b's `print`. The run-arm's skip logic
  keys on *whether a test needs an unregistered capability*, not on the bare
  `Stage::Run` scalar.
- **File length.** The machine will be large; split by concern from the
  start (heap/, machine/{step,call,unwind,tailcall,gc}, …) to stay under the
  hygiene limits — don't grow one `machine.rs` blob (CLAUDE.md).
- **Determinism.** No default-hasher `HashMap` on any Doodle-observable path;
  fixed-key SipHash; index-order sweep; no wall-clock; fixed float format
  (Ryū). Every determinism-gate diff is a release blocker.
