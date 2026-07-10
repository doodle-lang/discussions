# Doodle Machine Design (doodle-core internals)

**Status:** v0.2 — **accepted by the user 2026-07-10** (after adversarial
review and revision); the M2a gate is satisfied. Changing the *mechanisms*
pinned here still requires revising this document first (§ intro).
· **Date:** 2026-07-10

This document pins the load-bearing internal design of `doodle-core`'s
execution machine, refining the implementation plan's AD2 (CESK-style AST
machine), AD3 (index-slab heap), and AD5 (binding cells) into concrete
structures. It exists because these choices are deep and coupling-heavy:
letting the first implementing session make them ad hoc was identified as
the plan's riskiest gap for agentic execution.

v0.2 incorporates the findings of a hostile design review; the largest
change is §12 (unwinding), whose v0.1 "nearest-X" dynamic scans were shown
unsound — exit targets are now resolver-annotated and frames carry explicit
kind/consumer linkage. Review findings that *defended* the design (frame-
index stability, S-45's blanket form, suspension sufficiency, index-reuse
identity, clone termination) are recorded inline where they pin invariants.

Notation: **L** = language spec, **E** = engine spec, **plan** =
`implementation.md`. Rust type sketches are *shapes*, not final signatures;
renaming and field reshuffling during implementation is fine, changing the
*mechanisms* is not (that requires revising this document first).

---

## 1. Ground rules

1. **Safe Rust only** in `doodle-core` (plan AD1). Everything below is
   expressible with `Vec`, indices, and enums — no `unsafe`, no `Rc`
   cycles, no raw pointers in the value graph.
2. **Indices, not references.** All cross-structure links are typed `u32`
   indices (newtypes). This is what makes state snapshot-friendly,
   deterministic, and borrow-checker-tractable.
3. **Determinism** (plan §4.1): no `std::collections::HashMap` with the
   default hasher anywhere Doodle-observable; fixed-key SipHash;
   allocation-count-triggered GC; index-ordered sweep; no wall-clock
   anywhere in the core.
4. **The machine state is the observable state** (plan AD2): every E§8
   observation must be a direct read of the structures below — if a design
   change would require a reconstruction layer for observability, it is
   the wrong change.
5. **Lexical facts are resolved statically.** "Nearest enclosing X" for
   `break`/`continue`/`return` is a *lexical* property; the resolver
   annotates it. The machine only performs *dynamic* searches for the two
   genuinely dynamic questions: exception handlers and dispatch.

## 2. Compilation units: AST arena and resolved modules

The front end produces, per module, an immutable **`ResolvedModule`**:

```rust
struct ResolvedModule {
    canonical_id: ModuleId,          // from the host resolver (E§6)
    nodes: Vec<Node>,                // flat AST arena; NodeId(u32) indexes
    spans: Vec<Span>,                // parallel to nodes
    stmt_spans: Vec<(Span, NodeId)>, // statement-span index (breakpoints, E§8.6)
    callables: Vec<CallableInfo>,    // per to/fn/anon-fn/block body: params,
                                     //   slot tables, capture plan, docstring,
                                     //   tail-marked body, kind, exit annotations
    globals: Vec<GlobalDecl>,        // module-level names -> declared kinds
    name_refs: Vec<NameRef>,         // free-name reference sites (see below)
}
```

- **Flat arena, `NodeId(u32)`.** `Cont` entries and callable references
  hold `Copy` node ids, never Rust references — the whole machine state
  stays index-based (ground rule 2).
- `ResolvedModule` is **immutable** after load and lives outside the GC
  heap (in the instance's module table; `Arc`-shareable with tooling).
  Consequently the free-name **cell cache is per-instance, not in the
  module**: each instance keeps a `Vec<Option<CellIdx>>` parallel to
  `name_refs`, filled lazily on first execution (AD5). (v0.1 put the cache
  in `ResolvedModule`; that conflicts with immutability/sharing.)
- The resolver has already: assigned local **slots**, computed **static
  links** for block bodies (§7), marked captured locals as **cell-boxed**
  (§7), marked **tail positions** (§11), annotated every
  `break`/`continue`/`return` with its **lexical exit target** (§12), and
  attached statement boundaries (safe points, §9).

## 3. Values

```rust
#[derive(Clone, Copy)]
enum Value {
    Nil,
    Bool(bool),
    Int(i64),            // small-int fast path
    BigInt(BigIntIdx),   // heap bignum (num-bigint)
    Float(f64),
    Str(StrIdx),
    Bytes(BytesIdx),
    List(ListIdx),
    Dict(DictIdx),
    Record(RecIdx),      // value records AND ref records (header says which)
    Callable(CalIdx),    // to/fn/lambda; one canonical CalObj per plain to/fn (§8)
    Module(ModId),
    Type(TypeIdx),       // built-in type values + record types + protocols
    Foreign(FrnIdx),
}
```

- **No `derive(PartialEq)`.** Equality is the *semantic* function of
  L§4.13 (structural, cycle-safe, cross-numeric-kind), implemented
  explicitly; a derived bitwise `==` on `Value` would be a footgun and is
  deliberately absent.
- **Size refinement vs the plan.** Plan §3.4 says "8-byte tagged
  representation." A safe-Rust enum with an `f64`/`i64` payload is 16
  bytes; v0.1 **accepts the 16-byte `Copy` enum** and treats 8-byte
  packing as a §6.5-budget-driven optimization that must not change
  observable behavior.
- **Canonical-int invariant:** an integer representable as `i64` is
  *always* `Int`, never a `BigInt` — arithmetic promotes on overflow and
  demotes results that fit. This removes **Int↔BigInt** comparison from
  equality/hash paths. It does **not** remove cross-kind *numeric*
  equality: L§4.13 requires `1 == 1.0`, and S-28 requires exact
  comparison beyond 2⁵³ plus **hash coherence** (`hash(Int(1)) ==
  hash(Float(1.0))`, and likewise for equal BigInt/Float values), since
  numbers are dict keys. Design: numeric equality compares exact
  mathematical values (integer-vs-float comparison done exactly, not via
  lossy widening); numeric hashing hashes the *mathematical value* (an
  integral float in range hashes as its integer; otherwise a canonical
  float form). The precise S-28 semantics (NaN, −0.0) land with that
  delta.
- `Void` (the L§6.11 procedure-result sentinel) is **not** a `Value`
  variant: the machine's result register is `Option<Value>` with `None` =
  Void, so a Void can never be stored into a data structure by
  construction (§8).

## 4. Heap: per-kind slabs

Per instance, one slab per heap kind:

```rust
struct Slab<T> {
    slots: Vec<Slot<T>>,      // Slot = Occupied { obj: T, mark: bool, serial: u64 }
                              //      | Free { next_free: Option<u32> }
    free_head: Option<u32>,
    live_count: u32,
}
struct Heap {
    strings: Slab<StrObj>,     // NFC UTF-8 payload + grapheme memo (§5)
    bigints: Slab<BigIntObj>,
    bytes:   Slab<BytesObj>,
    lists:   Slab<ListObj>,    // Vec<Value>
    dicts:   Slab<DictObj>,    // insertion-ordered entries + SipHash index (fixed key)
    records: Slab<RecObj>,     // type: TypeIdx, fields: Vec<Value>, is_ref: bool
    callables: Slab<CalObj>,   // (ModuleId, CallableId) + captured cells: Vec<CellIdx>
    cells:   Slab<CellObj>,    // boxed upvalues (§7) AND module binding cells (§6)
    types:   Slab<TypeObj>,
    foreigns: Slab<ForeignObj>, // type_tag, host data, finalizer — own segment (AD3)
    bytes_allocated: u64,       // program-driven payload bytes (AD3; NOT caches, §5)
    alloc_serial: u64,          // monotonic; stamped on every allocation
}
```

- **Variable-size payloads** (string bytes, list/dict backings) are the
  objects' own Rust allocations, counted in `bytes_allocated` (plan AD3).
- **Identity** for reference-typed values is the slab index. This is
  sound even with host handles in play: a handle is a GC root, so an
  object the host still references is never swept and its index never
  reused while any witness to the old identity exists (review-defended).
  `serial` provides a deterministic identity-derived scalar where one is
  ever needed. `is_same` = same variant + same index.
- **Value records vs ref records** share `Slab<RecObj>`; the
  value/reference distinction lives entirely in *binding semantics*
  (L§4.14): binding/argument-passing a value record clones its `RecObj`
  (recursively cloning value-record fields, sharing everything else).
  **Acyclicity invariant (review-defended, must be documented and
  debug-asserted):** the value-record-contains-value-record graph is
  acyclic by construction — every write of a value record into any
  field/binding copies it atomically (S-38 place chains navigate without
  creating sharing), and a value record can reach itself only through a
  reference-typed hop, which the clone shares without descending. The
  cloner is deliberately not cycle-safe and relies on this; add a
  debug-only depth bound.

## 5. Strings

`StrObj { utf8: Box<str> /* NFC by construction */, graphemes: OnceCell<Box<[u32]>> }`
— the grapheme memo is byte offsets of cluster boundaries, built lazily on
first index/length request. **The memo is a pure cache and must be
invisible to determinism machinery**: its allocation is *not* counted in
`bytes_allocated` (which drives GC triggers and the heap limit), because
memo construction can be triggered by *host inspection* (E§8.4), and
observed vs unobserved runs must have identical GC trigger points
(E§7.7). All construction paths (literal, concat seam pass per plan AD4,
`make_string` decode) produce NFC.

## 6. Names: module binding cells (plan AD5)

Every module-level name binds a **cell** in the shared `cells` slab:

```rust
struct CellObj { kind: CellKind, value: Value }
enum CellKind {
    Let,                       // mutable module binding
    Const,
    Parameter,                 // dynamic parameter; `value` is the CURRENT
                               //   dynamic value (default installed at decl;
                               //   `with` swaps it, §13)
    Dispatcher(ProtoMemberId), // protocol member; dispatch on first arg
}
```

A module's namespace is `name -> Binding { cell: CellIdx, provenance }`
where provenance = `Declared | Selective(ModId) | Wildcard(ModId) |
WildcardAmbiguous(ModId, ModId)`. Wildcard imports **alias the exporter's
cells** (same `CellIdx`); imported bindings are **read-only aliases**
(S-39). Free names resolve through the per-instance name-ref cache (§2).

**Replay note:** native-module and prelude registration order shapes
initial cell/serial assignment, so it is host input that belongs in the
replay artifact's identity (S-36 adjacency) — record it.

## 7. Environments: slots, cells, and static links

The resolver classifies every local as:

- **slot-direct** — lives in a frame's `locals: Vec<Option<Value>>` at a
  fixed index; the common case; or
- **cell-boxed** — captured by a nested `fn` (closures own their captures
  beyond the frame's life, and may mutate them per S-11): the local's
  storage is a heap `CellObj`; the frame slot holds the cell reference and
  a `Callable`'s capture list references the same cells.

Second-class **blocks do not capture**: a block body reads/writes
enclosing locals **through the defining chain**. Because blocks nest, an
outer-local reference from a block body is resolved by the resolver to a
**static link**: `(hops, slot)` — chase the block frame's
`defining` link `hops` times, then index that frame's `locals` (hops = 0
is the block's own frame). A single-link "the defining frame" scheme is
wrong for nested blocks (`each(xs) do(x) each(ys) do(y) total = … end
end` reaches the fn's `total` two hops up); v0.2 pins classic static
links. Legal because a block cannot outlive its defining chain (L§8.5).

## 8. Frames and continuation stacks

```rust
enum FrameKind {
    Callable { cal: CalIdx },       // to/fn/lambda body; owns a ReturnBarrier
    Block {
        defining: FrameIx,          // static-link parent (§7)
        defining_serial: u64,       // integrity check
        consumer: Consumer,         // who received this block (for `break`, §12)
    },
    ModuleTopLevel { module: ModId },
}
enum Consumer {
    DoodleCall { frame: FrameIx, serial: u64 }, // non-tail Doodle consumer
    TailReused,                                 // consumer frame was reused (§11)
    Native { drive_depth: u32 },                // invoked via reentrant drive (§14)
}
struct Frame {
    kind: FrameKind,
    call_site: Span,
    locals: Vec<Option<Value>>,
    block_param: Option<BlockDescriptor>, // the bound `do body` parameter, if any
    conts: Vec<Cont>,                // this frame's pending work (LIFO)
    tail_count: u64,                 // E§8.3 per-frame tail-iteration counter
    serial: u64,                     // frame identity; PRESERVED across tail reuse
    dyn_depth: u32,                  // dynamic-parameter stack depth at entry (§13)
}
struct Machine {
    frames: Vec<Frame>,              // the walkable stack (E§8.2)
    reg: Option<Value>,              // result register; None = Void (L§6.11)
    unwind: Option<Unwind>,          // in-flight non-local transfer (§12) — GC root
    ring: RingBuffer<ElidedFrame>,   // tail-elided history (E§8.3); entries tagged
                                     //   with the consuming frame's serial (S-34)
    dyn_stack: Vec<(CellIdx, Value)>,// saved dynamic-parameter values (§13)
    pending: Option<CapabilityRequest>, // set while Suspended (§14)
    drive_stack: Vec<DriveCtx>,      // nested drives (§14)
    fuel: FusedCounter,              // §9
}
```

Notes forced by review:

- **`FrameKind::Callable { cal }` stores the invoked callable *value***
  (`CalIdx`), not just `(ModId, CallableId)`: E§8.2 exposes each frame's
  callable as a handle and traces carry callables, and callable equality
  is identity (L§4.9) — so the frame must hand back the *same* `CalObj`
  the program called. Plain `to`/`fn` declarations intern one canonical
  `CalObj` at module load; closures use their own.
- **`block_param`** is the storage for a bound block parameter (a `do
  body` argument): block descriptors are not `Value`s and cannot live in
  `locals` — v0.1 left them homeless. The resolver guarantees a block
  parameter name is only ever used in invocation position, preserving
  "never storable in Doodle data" by construction.
- **FrameIx stability (review-defended):** the frames Vec mutates only at
  the top, and a live block descriptor's defining frame is always parked
  deeper in the stack (at an `Apply`), so `FrameIx + serial` links cannot
  dangle in any legal execution.

**The `Cont` taxonomy** (defunctionalized pending work; representative —
variants are added as the grammar demands, but must fall in these
categories):

| Category | Examples | Notes |
|---|---|---|
| sequencing | `Seq { body, next }` | statement boundary = safe point |
| expression plumbing | `BinRhs{op,rhs}`, `BinApply{op,lhs}`, `UnaryApply`, `CollectList{..}`, `CollectDict{..}`, `Interp{..}`, `IndexApply`, `FieldGet` | hold `Value`s → conts are GC roots |
| calls | `EvalArgs{..}`, `Apply{..}` | argument binding per L§8.3 at `Apply`; `Apply` records whether a block argument was passed (consumer linkage, §12) |
| binding/assignment | `BindSlot{slot}`, `BindCell{cell}`, `PlaceStep{..}`, `PlaceAssign` | place chains per S-38 |
| control | `IfElse{..}`, `WhileReloop{node,cond,body}`, `LoopReloop{node,body}` | loop conts carry their `NodeId` so exits can target them (§12) |
| **cleanup (unwind-active)** | `WithRestore{dyn_mark}`, `TryHandler{binder, body}` | the ONLY variants unwinding executes (§12); `WithRestore` holds the `dyn_stack` index to pop to, not a duplicate value |
| markers | `ReturnBarrier` (callable body root), `HostBoundary` (nested drive, §14) | |

`step()` pops the top `Cont` of the top frame (or, when empty, returns
from the frame). Every transition consumes and/or produces `reg`; a
transition that uses `reg` as a value errors (L§6.11) if it is `None`.

## 9. Safe points and the fused counter

Safe points fire (E§7.4) at `Seq` boundaries, call entry, and return. One
decrement-and-branch per safe point on a single **fused counter** =
min(remaining slice fuel, remaining step budget,
distance-to-next-armed-event); reaching 0 enters the slow path (fuel →
`Paused(SliceEnd)` per S-40; budget → `Faulted(LimitExceeded)`;
breakpoint/step/pause/cancel per directive). Per-subexpression
observation mode adds safe points at expression-plumbing conts (a runtime
flag; the step *budget* counts only statement-level points, S-20).
**GC-trigger and heap-limit checks likewise fire only at statement-level
safe points** — never at fine-mode-only points — so observation mode
cannot move a collection or a limit fault (E§7.7).

## 10. Calls, blocks, closures

- **Call**: evaluate callee, evaluate args left-to-right, `Apply` binds
  per L§8.3, pushes a `Frame` (kind `Callable`) with a `ReturnBarrier`
  cont and the body `Seq`.
- **Block argument**: a *block descriptor* internal to the machine (never
  a Doodle value): `(defining FrameIx, defining serial, CallableId)`.
  It is bound into the callee frame's `block_param`. Invoking it pushes a
  `Block` frame whose `defining`/`defining_serial` come from the
  descriptor and whose `consumer` records **who invoked it**: the
  invoking Doodle frame (`DoodleCall`), `TailReused` when the consumer
  tail-invoked it (§11), or `Native{drive_depth}` when a foreign function
  invoked it through a reentrant drive (§14). Outer-local access uses the
  §7 static links. The descriptor's serial check at the drive-layer
  boundary (native consumers hand back an invocation token) is
  **always-on**, not debug-only — the host is untrusted input.
- **Closures (`fn` values)**: `CalObj` holds the capture cell list;
  entering the closure splices those cells into the new frame. `return`
  inside a closure is local to it (L§6.10).

## 11. Proper tail calls

The resolver marks a call node *tail* iff its value is the enclosing
callable's result with no pending work, **and** it is not inside a
`with`/`try` body (L§8.7), **and** it does not pass a block argument
(**S-45**, resolved at M1: since L§8.5 forbids forwarding a received
block, the only passable block is a literal `do … end` referencing the
*current* frame — so the blanket exclusion is exactly the necessary set;
if block forwarding is ever legalized, S-45's form must be revisited).

Executing a marked tail call: push `(callable, call_site, span,
consuming_frame_serial)` of the current occupant into `ring` (S-34), then
overwrite the frame **completely**: `kind` (including `defining`/
`consumer` when a callable tail-invokes its own block parameter — the
reused frame *becomes* a `Block` frame with `consumer: TailReused` — and
conversely a block frame tail-calling a plain fn becomes `Callable`),
`call_site`, `locals`, `block_param`, `conts`; `serial` and the logical
depth are **preserved** (E§8.5 stepping = frame-identity comparison);
`tail_count += 1`. v0.1 under-specified the overwrite set; stale
`kind`/`block_param` corrupts §12's target resolution.
`debug_assert!(dyn_stack.len() == frame.dyn_depth)` at reuse — the
tail-marking rule implies no `with` is open in this frame
(review-defended, but a resolver bug here would corrupt dynamic
parameters silently, so assert it).

## 12. Unwinding: one mechanism, resolver-annotated targets

All non-local transfers are represented by a single in-flight record:

```rust
enum Unwind {
    Continue { value: Option<Value> },              // target: annotated
    Break    { value: Option<Value> },              //   (see below)
    Return   { value: Option<Value> },
    Raise    { exception: Value, trace: TraceIdx }, // trace captured at raise
    Cancel,
}
```

`Machine.unwind` (with its exception value and trace) **is a GC root**
while a transfer is in flight — including for the duration of a rescue
body, so bare `raise` can re-raise with the original trace (v0.1 omitted
this root).

The unwinder pops conts (and frames when a frame empties), **executing
only** `WithRestore` (restore the dynamic parameter) and — for `Raise`
only — `TryHandler` (bind the exception, enter the rescue body, clear
`unwind`). All other conts, including `TryHandler` during
non-raise unwinds, pop inertly. What distinguishes the exits is their
**target**, and targets are *lexical*, annotated by the resolver on every
`break`/`continue`/`return` node (ground rule 5 — v0.1's dynamic
"nearest-X" scans were unsound):

- **`continue` → `ThisLoop(NodeId)` or `ThisBlock`.** In a loop: pop the
  current frame's conts to the matching `WhileReloop`/`LoopReloop` cont
  (matched by NodeId) and re-enter it. In a block body: finish this block
  invocation, delivering the optional value to the invoker (the ordinary
  block-return path).
- **`break` → `ThisLoop(NodeId)` or `ConsumerCall`.** In a loop: pop to
  the matching loop cont and discard it, resuming after the loop (the
  value question is S-10). In a block body: consult the current `Block`
  frame's `consumer`:
  - `DoodleCall{frame,serial}` — unwind frames from the top through the
    consumer frame inclusive (executing `WithRestore`s throughout);
    deliver the optional value as the consumer *call's* result to the
    pending cont beneath it;
  - `TailReused` — the consumer's continuation *is* this frame's: finish
    the reused frame, delivering the value as its result;
  - `Native{..}` — park at the drive boundary; see below.
- **`return` → `HomeCallable`.** Chase the current frame's
  defining chain (0+ hops for nested blocks) to the lexically enclosing
  `Callable`/`ModuleTopLevel` frame — **not** "the nearest ReturnBarrier
  above" (v0.1's rule wrongly returned from the block's *consumer*, e.g.
  exiting `each` instead of the function that wrote the `return`).
  Unwind everything from the top down through that frame, delivering the
  value (or Void for a `to`) as its result. Intervening frames may
  include Doodle consumers (unwound normally) or native consumers
  (parked, below).
- **`raise`** searches *dynamically* for the nearest `TryHandler` — the
  one genuinely dynamic target — capturing (exception, trace incl. ring
  snapshot) at the raise site before any unwinding, with the raise-trap
  pause (E§8.7) at that same point for all three raise sources (S-18).
- **cancellation** unwinds everything (running `WithRestore`s, skipping
  `TryHandler`s), under S-23's reserved-budget rules.

**S-9 note (proposed resolution).** This mechanism makes `break`/
`continue` punch *through* an intervening `with` body — executing its
restore — to reach their lexical loop/consumer target. That contradicts
L§7.10's current "simply exit the with body" wording and is the behavior
S-9 was filed to fix (loop control from inside `with` is otherwise
impossible); this design is the proposed S-9 resolution and must land as
the S-9 spec edit no later than M2a.

**Non-local exits through a native consumer (files S-46).** When the
unwinder reaches a `HostBoundary` (the exit must cross a reentrant
drive): park the `Unwind` (it stays rooted) against the drive-stack
entry, and the nested drive returns a distinguished outcome
`NonLocalExit { kind }` to the foreign callback, which **must** return to
the engine promptly without producing a result. When control returns to
the foreign call's `Apply`, the engine validates the parked token and
resumes: a `Break` targeting *this* native call completes the `Apply`
with the value; anything else continues unwinding outward. A host that
instead returns a value or raises after `NonLocalExit` faults the
instance. v0.1 cited S-16 for this; S-16 covers *abandoned* drives —
this protocol is new engine-spec surface (E§7.2 outcome variant + E§5.4
host obligations), filed as **S-46** (resolve by M2b).

## 13. Dynamic parameters

`parameter` declares a `Parameter` cell holding the current value.
`with p = v do … end`: evaluate `v`, push `(cell, old_value)` onto
`dyn_stack`, write `v` into the cell, push `WithRestore { dyn_mark }`
(the index to pop to — the saved value lives only in `dyn_stack`, no
duplication); restore on normal completion or any unwind (§12). Reads
are one cell load. Frames record `dyn_depth` so E§8.2 can report which
bindings each frame established.

## 14. Suspension and the drive stack

A capability call reaches `Apply` with a suspending-capability callee:
the machine stores `CapabilityRequest { capability_id, args }` in
`pending` and the drive loop returns `Suspended`. **No state is torn
down**; observation works as at a safe point (S-17). Sufficiency is
review-defended: every partial result of an enclosing expression lives in
defunctionalized conts, never in `reg`, so `reg`-in + park is complete.
`resolve(Value(v))` sets `reg` and continues; `resolve(Raise(e))` enters
§12's raise path at the call site (trace captured pre-unwind; raise-trap
per S-18).

**Nested drives** (E§5.4/§7.6): a synchronous foreign function invoking a
Doodle callable (or a received block, via its invocation token — §10)
pushes `DriveCtx { boundary: FrameIx, directive, parked: Option<Unwind> }`
and a `HostBoundary` cont. Arguments passed to a synchronous foreign
callback are **rooted for the callback's duration** (they are reachable
from the suspended `Apply` cont and, for the C ABI, from the temporary
argument handles). Suspending capabilities under a nested drive remain
the S-15 question; the mechanism supports both candidate resolutions.

## 15. Garbage collection

- **Trigger:** only at *statement-level* safe points (§9), only by
  `bytes_allocated` crossing a threshold — replay-identical and
  observation-mode-independent.
- **Roots:** handle table (§16); every frame's `locals`, `block_param`
  (via its defining-frame link — no heap refs, but listed for
  completeness), and `conts` (values embedded in cont variants); `reg`;
  `Machine.unwind` (exception + trace, §12); `dyn_stack` saved values;
  module namespaces **including `Loading`-state partial namespaces and
  `Failed(ErrorIdx)` retained exception values** (§18 — v0.1 omitted the
  latter; re-raise on re-import must not resurrect a recycled object);
  `pending` request args; `ring` entries' callable refs; native/prelude
  registrations; arguments held by in-flight synchronous foreign calls
  (§14).
- **Mark:** explicit work-stack (no recursion — cycles are legal).
- **Sweep:** each slab in **index order**, rebuilding its free list in
  index order (plan AD3). Foreign finalizers queue during sweep and run
  host-side after collection completes (never Doodle-observable).
- **Heap limit:** checked against `bytes_allocated` (which excludes pure
  caches, §5) after a failed collect → `Faulted(LimitExceeded(heap))`.

## 16. Handles

`HandleTable: Vec<HandleSlot { value: Value, gen: u32, refs: u32 }>` with
a free list; a handle is `(index: u32, gen: u32)` packed in `u64`
(+ instance id in debug builds). `retain`/`release` adjust `refs`.
Handles are GC roots. Generation checks happen at the host boundary
(always-on — host input is untrusted); the machine's internal indices
never pay them.

## 17. Observability mapping (E§8 ⇄ structures)

| E§8 requirement | Where it reads from |
|---|---|
| current position (8.1) | span of the active cont's NodeId; **at a return safe point, the position is the just-executed call's span** (pinning E§8.1's "about to execute or just executed" for the return case) |
| frames, callable, call-site (8.2) | `frames[i]`; the callable handle is the frame's `CalIdx` — identity-correct for closures (§8) |
| named locals (8.2) | `locals` + the callable's resolver slot-name table |
| dynamic bindings per frame (8.2) | `dyn_stack` sliced by frames' `dyn_depth` |
| tail history + counters (8.3) | `ring` (entries carry consuming-frame serials, S-34) + `tail_count` |
| pure value inspection (8.4) | direct slab reads; never runs `step()`; may build the §5 memo (deliberately unaccounted) |
| stepping depth rules (8.5) | frame identity (`FrameIx` + `serial`); **if a step anchor's frame is popped by a non-local unwind, the directive completes at the next safe point after the unwind settles** (anchor-invalidation rule; v0.1 was silent) |
| breakpoints (8.6) | `stmt_spans` index; pending breakpoints keyed by module id |
| raise-trap (8.7) | the single raise entry point of §12 |
| eager/lazy local capture (8.8) | moot: locals are always live |

**Auxiliary evaluation caveat (S-22 adjacency):** driving `to_string` on
a paused instance is an ordinary drive — it allocates, advances serials,
and can run GC. It therefore perturbs the heap relative to an
un-inspected run. Replay identity (E§11) is defined over *program-driven*
execution; a recorded run must either forbid auxiliary evaluation or
record it as part of the resolution stream. This lands with S-22.

## 18. Module runtime

Instance module table: `ModuleId -> { state: Unloaded | Loading | Loaded |
Failed(ErrorIdx), namespace, resolved: Arc<ResolvedModule> }`. `import`
executing mid-drive on an `Unloaded` module invokes the host resolver
(E§6), parses/resolves, then pushes the module top-level as an ordinary
`ModuleTopLevel` frame (observable, suspendable); `Loading` hit again =
circular-import raise; a raise during load ⇒ `Failed` retaining the
exception (rooted, §15), re-import re-raises (S-8). Native modules occupy
the same table (S-32/S-44 govern their contents; registration order is
replay-identity input, §6).

## 19. Open questions for the M2a implementer

Deliberately not pinned here (small, local, or awaiting S-items):

- Exact `Cont` variant list (categories pinned in §8).
- Small-int/`BigInt` promotion helper API; float formatting entry point
  (Ryū); the S-28 NaN/−0.0 decisions.
- Dict tombstone/compaction policy (must preserve insertion order and
  determinism; otherwise free).
- `FusedCounter` re-arm bookkeeping when events arm/disarm mid-run.
- S-15 (nested-drive suspension) — drive layer decides after M3
  prototyping; §14's mechanism supports both candidates.
- The `TraceIdx` representation (heap object vs side table) — must be
  GC-rooted via `unwind` and exception values either way.

## 20. Refinements vs the implementation plan (to fold back)

1. `Value` is a 16-byte safe enum for v0.1 (§3), not the plan's "8-byte"
   phrasing — packing is a later, behavior-invisible optimization.
2. **S-45** (calls passing a block argument are not tail): filed in plan
   App C, resolved at **M1** (tail marking), consumed at M2a (§11).
3. **S-46 (new, from v0.2 review):** non-local-exit-through-native-
   consumer protocol — `NonLocalExit` outcome + host obligations (§12);
   needs an App C entry, resolve by M2b.
4. **S-9:** §12's punch-through-with-restoration is the proposed
   resolution; the L§7.10 edit lands by M2a.
5. Upvalue cells and module binding cells share one `cells` slab (§6/§7).
6. Free-name cell caches are per-instance, not in `ResolvedModule` (§2).
7. Grapheme-memo allocations are excluded from `bytes_allocated` (§5);
   GC/limit checks pinned to statement-level safe points (§9, §15).

## 21. Change log

- **v0.2 (2026-07-10):** post-review revision. Rewrote §12 around
  resolver-annotated exit targets + frame kind/consumer linkage (four
  blockers: return punch-through target, loop exits unrepresentable,
  consumer scan unsound under tail invocation, nested-block static
  links). Added: `block_param` storage, callable-identity in frames,
  complete tail-reuse overwrite set, `unwind`/`Failed`/sync-args GC
  roots, memo/GC-trigger determinism fixes, numeric equality/hash
  correction, S-46 filing, S-9 proposal, return-position and
  step-anchor-invalidation rules, aux-eval caveat, always-on boundary
  checks, per-instance name caches, replay note for registration order.
- **v0.1 (2026-07-10):** initial draft.
