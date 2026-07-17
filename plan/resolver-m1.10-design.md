# M1.10 Resolver — design proposal (pending ratification)

**Status:** proposed, awaiting the user's ratification. This concretizes the
resolver output that `machine-design.md` (**MD**) §2/§6/§7/§11/§12 specifies only
as prose/comments, and proposes resolutions for the two gating spec deltas (S-5,
S-6). Once ratified, the concrete types below fold into MD (or stay in code
citing MD, as the AST does), and S-5/S-6 land in the language spec + Appendix C.

The resolver is a single pass over the parsed `Ast` producing an immutable
`ResolvedModule`. Nothing consumes it yet (the M2a machine is the first
consumer), so it can be built and reshaped in isolation.

---

## 1. What is already upstream (not resolver work)

From the M1.10 requirements, these are **already emitted** and the resolver need
not re-implement them (it may assume a clean-enough tree or coexist):

- **Chained comparison** — parser (`parse.rs`, `ChainedComparison`).
- **Surrogate escapes** — lexer (`lex/escape.rs`, `MalformedEscape`).
- **Module-level-only placement** (`record`/`protocol`/`implement`/`module`/
  `parameter`/`exports`/`import` nested in a body) — parser (`require_module_level`).
- **Non-lvalue assignment target** (`1 = 2`) — parser (`SyntaxError`).

New-to-resolver: scopes/slots, capture classification, block static links,
free-name classification (`name_refs`), the exit-target annotations, tail
marking, and the error classes **const-reassignment**, **duplicate-declaration**,
**undeclared-name assignment** (assigning a name bound nowhere — local or module —
where lexically determinable; L§5.3), **exit placement**, and
**fn-falls-off-end**. (Imported-**alias** assignment — a *read-only* alias
target, S-39 — is a distinct, separately-deferred category needing import
resolution: M5, not M1.10.)

---

## 2. Concrete `ResolvedModule` types (proposed)

Faithful to MD §2 (the `ResolvedModule` struct is quoted there verbatim). The
gaps MD leaves — `CallableInfo`, `GlobalDecl`, `NameRef`, the slot table, the
capture plan, and the exit-target annotation — are filled below. **Every gap-fill
is flagged `[DECISION]` for ratification.**

```rust
/// The resolver's per-module output: the AST arena plus the resolved
/// environment (MD §2). Immutable after load, Arc-shareable, lives outside the
/// GC heap. The per-instance free-name cell cache (Vec<Option<CellIdx>> parallel
/// to `name_refs`) lives in the instance, NOT here (MD §2, AD5).
pub struct ResolvedModule {
    pub canonical_id: ModuleId,       // from the host resolver (E§6); ModuleId(0) at M1
    pub ast: Ast,                     // the flat arena (owns nodes+spans); NodeId(u32) indexes
    pub root: NodeId,                 // the Node::Module root
    pub stmt_spans: Vec<(Span, NodeId)>, // statement-span index (breakpoints, E§8.6)
    pub callables: Vec<CallableInfo>, // one per to/fn/anon-fn/block body
    pub globals: Vec<GlobalDecl>,     // module-level names -> declared kinds
    pub name_refs: Vec<NameRef>,      // free-name reference sites
    pub resolutions: Vec<Option<Resolution>>, // parallel to ast nodes; Some(_) at each
                                     //   Ident ref AND each local decl (Let/Const/Param)
}
```

**[DECISION 1] `resolutions` as a side table.** MD says the resolver "annotates
names … on top of this tree." Rather than mutate `Node::Ident` to carry a
resolution, this proposes a `Vec<Option<Resolution>>` indexed by `NodeId`
(parallel to the arena, index-based per ground rule 2). Both `Ident` *reference*
sites **and local *decl* sites** (`Let`/`Const`, each `Param`) get a `Some`: a
`let x = v`'s binding slot lives on the `Let` node (its name is a `Node` field,
not an `Ident` child), so the machine's `BindSlot` cont reads the slot from
`resolutions[let_node]`. Alternative: a dense `Vec<Resolution>` keyed by a
separate "ident id", or storing the resolution inline in a rewritten node. The
side table keeps `Node` unchanged and is O(1) to consult from the machine.
*(Recommended: side table, covering decl + reference sites.)*

```rust
/// How a bare-name `Node::Ident` reference site resolves (MD §6/§7).
pub enum Resolution {
    /// A local in the *current* callable frame at `slot` (the common case). If
    /// that frame's `cell_boxed[slot]` is set, the machine derefs the CellObj;
    /// otherwise the slot holds the value directly.
    LocalSlot(u16),
    /// An enclosing local reached from a block body through the defining chain:
    /// chase `defining` `hops` times (hops=0 is the block's own frame), then
    /// index `locals[slot]` (MD §7 static links). Blocks do not capture; the
    /// target frame's `cell_boxed[slot]` still decides deref-vs-direct.
    BlockOuter { hops: u16, slot: u16 },
    /// A local captured by a nested `fn` closure: a heap cell in the closure's
    /// own capture list at `capture_index` (MD §7). The owning frame's
    /// `cell_boxed[slot]` was set at the same time.
    Capture(u16),
    /// A free name — resolved to a module binding cell lazily on first execution
    /// via `name_refs[name_ref]` and the per-instance cache (MD §6, AD5).
    ModuleName(u32),
}
```

**[DECISION 2] `Resolution` variant set.** These four are exactly MD §6/§7's
categories (slot-direct / block static link / cell-boxed capture / module cell).
Own-frame access to a *cell-boxed* local is **not** a fifth variant — it stays
`LocalSlot`/`BlockOuter`, with the owning frame's `cell_boxed[slot]` flag telling
the machine to deref. Dynamic parameters (`parameter` names) read as `ModuleName`
(their cell has `CellKind::Parameter`, MD §6) — a dynamic-parameter *read* is a
module-cell read; `with` swaps the cell value at runtime (§13); rule-2a's "no `=`
to a parameter" check reads `GlobalDecl.kind`, not the `Resolution`, so nothing is
lost by the shared variant. **Widths:** `u16` slot/hops, `u32` name-ref index; the
`usize → u16` slot conversion is **checked** and errors on overflow (a frame with
>65535 locals — pathological — must fail loudly, never wrap into an aliased slot;
mirrors `ast.rs`'s `u32::try_from(...).expect(...)`).

```rust
/// Per callable/block body (MD §2 `callables`). Grown across M1.10a→c: the
/// capture/exit/tail fields land with their chunks.
pub struct CallableInfo {
    pub kind: BodyKind,          // Proc | Func | Block | ModuleTopLevel
    pub decl: NodeId,            // the Callable/BlockArg/Module node
    pub body: NodeId,            // the Node::Block
    pub params: Vec<ParamInfo>,  // ordinary/block params, in order, each with its slot
    pub slot_count: u16,         // frame `locals` length to allocate
    pub slot_names: Vec<Box<str>>, // slot -> local name, all slots (params + body
                                 //   locals); MD §17's named-locals table (E§8.2)
    pub cell_boxed: Vec<bool>,   // per-slot: is this local cell-boxed? (MD §7). Set
                                 //   when a nested `fn` captures the slot; own-frame
                                 //   refs stay LocalSlot/BlockOuter and the machine
                                 //   derefs the CellObj when this flag is set. This
                                 //   is what makes the single forward pass sound —
                                 //   a late promotion flips one bit, leaving
                                 //   already-emitted references valid.
    pub doc: Option<Span>,       // docstring (L§8.6)
    pub captures: Vec<CaptureSource>,  // cells this closure captures (MD §7).
                                       //   M1.10a assigns indices + sets cell_boxed
                                       //   on the OWNER frame; CaptureSource SHAPE
                                       //   (the machine wiring) finalizes in M1.10c.
    // --- M1.10b ---
    // pub exits: Vec<(NodeId, ExitTarget)>,  // return/break/continue annotations (MD §12)
    // --- M1.11 ---
    // pub tail_sites: Vec<NodeId>,           // marked tail positions (MD §11)
}

pub enum BodyKind { Proc, Func, Block, ModuleTopLevel }

pub struct ParamInfo {
    pub name: Box<str>,
    pub slot: u16,
    pub is_block: bool,          // the trailing `do name` block parameter (§8.2)
    pub has_default: bool,       // default expr lives in the AST at the Param node
}
```

**[DECISION 3] `CallableInfo` grows by chunk.** M1.10a populates
kind/decl/body/params/slot_count/doc **and `captures`** (classification — its
`CaptureSource` shape finalizes M1.10c); `exits` (M1.10b) and tail marks (M1.11)
are added when their chunk lands. Alternative: define all fields now with empty
defaults. *(Recommended: grow per chunk — each field arrives with the code that
computes it and its tests.)*

```rust
/// A module-level declaration (MD §2 `globals`): a name and the cell kind it
/// binds (MD §6 `CellKind`). The cell itself is created at load; this records
/// the static declaration.
pub struct GlobalDecl {
    pub name: Box<str>,
    pub kind: GlobalKind,        // Let | Const | Parameter | Proc | Fn | Record | Protocol | Module
    pub decl: NodeId,
}

/// A free-name reference site (MD §2 `name_refs`): keys the per-instance cell
/// cache. `site` lets a diagnostic point at the use; `name` is looked up in the
/// module namespace on first execution.
pub struct NameRef {
    pub name: Box<str>,
    pub site: NodeId,
}
```

**[DECISION 4] `NameRef` fields.** MD's `name_refs` "(see below)" never lands on
a struct. Minimum viable: `{ name, site }` — enough to look up the binding and to
diagnose. The per-instance `Vec<Option<CellIdx>>` cache is parallel to
`name_refs` by index. *(Recommended as-is.)*

Reused from MD §6 (no change): `CellObj`/`CellKind`, `Binding`/provenance. Those
are load/runtime structures (M2a), not resolver output. **`GlobalKind` and
`CellKind` are distinct and not 1:1:** `GlobalKind` is the *declaration category*
the resolver records (for duplicate-detection, `const`-reassignment, and
diagnostics); the load step (M2a) maps it to a `CellKind` — `Let`→`Let`,
`Const`/`Proc`/`Fn`/`Record`/`Protocol`/`Module`→`Const` (all immutable module
bindings), `Parameter`→`Parameter`. Protocol **members** are a separate case:
they are not top-level `GlobalDecl`s but bind `CellKind::Dispatcher` cells
registered when the `protocol`/`implement` loads (M5) — out of M1.10 scope.

**Sketched for later chunks (not M1.10a):**

- **`ExitTarget`** (M1.10b, MD §12): `enum ExitTarget { ThisLoop(NodeId),
  ThisBlock, ConsumerCall, HomeCallable }` — `continue`→`ThisLoop`/`ThisBlock`,
  `break`→`ThisLoop`/`ConsumerCall`, `return`→`HomeCallable` (chase the defining
  chain to the enclosing Callable/ModuleTopLevel — **not** the nearest barrier).
  `raise` is **not** annotated (dynamic). **[DECISION 5, deferred]** whether
  `HomeCallable` needs an explicit hop count (MD §12 says "0+ hops" but gives no
  field) — reconcile with §7's `(hops, slot)` and §8's `Frame.defining`.
- **`CaptureSource`** (M1.10c, MD §7): how a closure names the enclosing cell it
  captures — `{ from_parent_slot: u16 }` or a parent capture index, for the
  machine to wire cells at closure creation. **[DECISION 6, deferred]** exact
  shape; depends on the M2a closure-creation path.

---

## 3. Resolver algorithm (single pass)

A recursive walk over **two distinct axes** (conflating them is the classic
resolver bug):

- **Lexical scope** — name *visibility*. **Every** construct body is its own
  scope (L§5.4): callable bodies, `do … end` block bodies, **and** `if`/`while`/
  `loop`/`with`/`try` arm bodies. Shadowing (M1.11) and duplicate-declaration are
  per lexical scope; a name resolves by walking scopes outward.
- **Frame** — *storage* (the runtime `locals: Vec<Option<Value>>`). Only
  **callable** bodies and **block** bodies open a frame (MD §8: each `FrameKind`,
  including `Block`, carries its own `locals`). A **construct body runs in the
  enclosing frame** (MD §8: `if`/`while`/… are `IfElse`/`WhileReloop`/… conts in
  the same frame, not frames), so a local declared inside a construct body takes
  a slot in the *enclosing* callable/block frame — its own lexical scope, shared
  slot space.

Sketch:

1. **Module top level** is a `CallableInfo{kind: ModuleTopLevel}`. Declarations
   **directly** in the module body (`let`/`const`/`to`/`fn`/`record`/`protocol`/
   `module`/`parameter`) become `globals` (module cells), *not* slots — visible
   module-wide (a `to` may reference a `let` declared later). But a `let` nested
   in a module-level **construct** body (`if c then let x … end`) is scoped to
   that body (L§5.4), so it is a `ModuleTopLevel`-frame **slot**, not a global —
   the same rule as step 4, applied at the top level.
2. **Entering a callable** (`to`/`fn`/anon-`fn`) opens a new lexical scope **and a
   new frame**; params take slots `0..n`; `let`/`const` in the body (and in
   construct bodies nested in it) take subsequent slots in *this* frame.
3. **Entering a block body** (`do … end`) opens a new lexical scope **and a new
   frame** (hops=0 is the block's own frame, MD §7); its own locals take slots in
   *its* frame; references to an enclosing frame's locals become static links.
4. **Entering a construct body** (`if`/`while`/`loop`/`with`/`try` arm) opens a
   new lexical scope **in the same frame**; its `let`/`const` take slots in the
   enclosing frame.
5. **Declaring a local** (`let`/`const`/a `Param`) assigns the next slot in the
   current frame, appends its name to that frame's `slot_names`, pushes `false`
   to `cell_boxed`, and records a `LocalSlot(slot)` in `resolutions` **at the decl
   node** (so the machine's `BindSlot` cont has the slot).
6. **Resolving an `Ident` reference** walks lexical scopes outward, counting the
   frame boundaries crossed:
   - found without crossing a frame boundary → `LocalSlot(slot)`;
   - found after crossing only **block** frame boundaries (`k` of them), no `fn`
     boundary → `BlockOuter{hops: k, slot}` (static link, MD §7 — legal because a
     block can't outlive its defining chain);
   - found after crossing a **`fn`** frame boundary → **set `cell_boxed[slot] =
     true` on the owning frame** (a late promotion — references already emitted as
     `LocalSlot`/`BlockOuter` stay valid, the flag drives deref at run time) and
     resolve this reference to `Capture(index)`; the closure's capture list
     records the source cell (multi-level capture and the exact `CaptureSource`
     wiring: [DECISION 6], M1.10c);
   - not found in any scope → a **free name** → append to `name_refs`, resolve to
     `ModuleName(idx)`.
7. **Assignment targets** resolve the same way; a bare-name target that resolves
   to nothing (no local scope, no `global`) is the **undeclared-assignment** error
   (§4); a target resolving to a `const` **or a declaration binding** (`to`/`fn`/
   `record`/`protocol`/`parameter`, S-6 rule 2a) is the **const-reassignment**
   error; only a `let`/`Let` target (local or global) is assignable.
8. **`slot_count`** per frame is the high-water mark of slots assigned in it
   (including construct-body locals, excluding nested callable/block frames);
   `slot_names`/`cell_boxed` have that length.

---

## 4. S-5 — the "static where determinable" boundary

> **RESOLVED (user, 2026-07-16). `implementation.md` Appendix C S-5 is now
> normative** — this section is the superseded proposal, kept for context. Ratified
> deltas to note when implementing: loop divergence = **no `break` lexically bound
> to the loop** (an exit-target-annotation lookup, **not** the reachability pass
> item 5/[DECISION 7] below proposed); the value-less list **includes `with`**;
> branch rule = any value-less→error, else any indeterminate→runtime, else OK
> (produces-or-diverges mixes fine); condition-blind; `fn` bodies only; tail-`while`
> diagnostic suggests `loop`.

**Goal (S-5):** pin the exact set of errors M1 rejects *statically* so every
conforming implementation rejects the same programs. Proposed normative list —
the resolver rejects statically iff **lexically determinable**:

1. **Duplicate declaration** — two `let`/`const`/`to`/`fn`/`record`/`protocol`/
   `parameter`/`param` bindings of the same name in the same scope. Always
   determinable → always static.
2. **`const` reassignment** — an assignment whose target resolves to a `const`
   binding visible in lexical scope. Static when the target is a bare name
   resolving to a lexical const; a field/index assignment through a const is a
   *mutation of the pointee*, not reassignment, and is allowed.
3. **Undeclared-name assignment** (L§5.3) — an assignment `x = …` whose bare-name
   target resolves to **no** binding (no local scope, no `global`). Static where
   determinable; a module-level name declared *later* in the same module is still
   a binding (module decls are visible module-wide), so only a name bound nowhere
   is rejected. (Distinct from the M5 imported-alias case, S-39.)
4. **Exit placement** — `return` outside any callable; `break`/`continue` outside
   any loop/block. Always lexically determinable → static.
5. **fn-falls-off-end** — a `fn` whose body can **complete without producing a
   value and without diverging**. The tail statement is classified:
   - **produces a value** (OK): a value-producing expression, `return expr`, or a
     tail `if`/`try` where *every* branch (including a present `else`) produces a
     value.
   - **diverges** (OK — never falls off): `raise`, a `loop` with no reachable
     `break`. A tail whose value comes from a diverging path is fine.
   - **value-less** (→ error, determinable): tail is `let`/`const`/assignment/
     `while`/bare-`return`/a `loop` that can `break`, or a tail `if`/`try` with a
     value-less branch or a **missing `else`**.
   - **indeterminate** (→ runtime, S-6): tail is a call whose proc/fn nature isn't
     lexically known (needs dispatch, M5).
6. Already-static upstream: chained comparison, surrogate escapes, placement,
   non-lvalue target.

**[DECISION 7]** the fn-falls-off-end classification (item 5) is the crux of S-5.
The subtle cases are the divergence ones (`raise`, no-`break` `loop`) — a naive
"tail is an exit/loop statement → value-less" rule (an earlier draft's bug) wrongly
rejects `fn f() raise Oops() end`. Proposing: classify the tail as
produces/diverges/value-less/indeterminate as above; reject only the *value-less*
case statically, defer *indeterminate* to runtime (S-6). The "no reachable
`break`" divergence check is the one place needing a small reachability pass —
confirm the scope (or simplify to "any `loop` tail is indeterminate → runtime").

---

## 5. S-6 — Void semantics

> **RESOLVED (user, 2026-07-16). `implementation.md` Appendix C S-6 is now
> normative** — this section is the superseded proposal. Ratified additions to note
> when implementing: producer-site **blame** in the diagnostic (span on the
> Void-producing expr); Void = MD §8's `None` result register, **propagates through
> expression-position `if`/`try`/parens** to the outer consumer and **never crosses
> an `fn` boundary**; the site list gains **`return`/`raise`/`break`/`continue`
> operands, parameter defaults (per-param + module `parameter`), keyword args,
> constructor calls, dict keys *and* values**; and the load-bearing **companion
> rule 2a** — declaration bindings (`to`/`fn`/`record`/`protocol`/`parameter`) are
> **non-assignable** (a new M1.10b battery member, folded into §3 step 7).

**Goal (S-6):** a block/`fn` whose last statement is value-less yields **Void**;
using Void where a value is required errors at the **consuming site**, uniformly
(amends §8.4's "raises at runtime" and §8.5's block-value sentence).

**Proposal:** `Void` is the engine sentinel for "no value" (procedure-call
results, non-value statements — MD §3/AD). Every value-consuming site rejects
`Void` with the L§6.11 diagnostic:

- `let`/`const` initializer, assignment RHS, call argument, operator operand, the
  **base/object of a field access (`x.f`) or index (`x[i]`)** and the index/key
  expression itself, interpolation expression, `if`/`while` condition, `with`
  value, collection element, and a `fn`'s result where consumed. (In short: every
  position except a *statement-position* expression, whose value is discarded.)
- **Static** where both the consuming site and the Void-producing tail are
  lexically determinable (e.g. `let x = p()` where `p` is a lexically-visible
  `to`); **runtime** otherwise, with the same diagnostic text.

This makes "procedure call in expression position" (§6.11) one instance of a
single uniform rule, rather than a special case.

**[DECISION 8]** whether Void-rejection is *only* runtime at M1 (the resolver
lacks type info to know an arbitrary call's proc/fn nature until M5 dispatch is
in) with just the lexically-obvious cases static, or whether M1.10 attempts more.
Proposing: static only for the lexically-obvious `to`-call-in-value-position
cases; everything else runtime at M2a. (The runtime conformance tests land at
M2a per the plan; M1.10 lands the static subset + the S-6 spec text.)

---

## 6. Chunk plan

- **M1.10a — environment pass (complete name resolution).** The `resolve`
  module; the types in §2 (exits/tail deferred); scope construction (all three
  scope kinds, §3); slot assignment; the **full** `Ident` classification —
  `LocalSlot`/`BlockOuter`/`Capture`/`ModuleName` — including block **static
  links** and **capture/cell-boxing marking** (a cross-`fn` reference is `Capture`
  and marks the target cell-boxed), so name resolution is *correct and complete*
  in one chunk (a partial pass would misresolve block/capture refs); `globals`;
  `name_refs`; `stmt_spans`. The `CaptureSource` wiring shape ([DECISION 6]) may
  stay a thin placeholder until M1.10c. No stage-gate bump. Unit tests over the
  `ResolvedModule` output.
- **M1.10b — static-error battery + exits.** duplicate-decl, const-reassign,
  undeclared-assignment, exit placement; the `ExitTarget` enum + annotations
  (MD §12). Diagnostic-code additions.
- **M1.10c — Void + falls-off-end.** S-5 fn-falls-off-end, S-6 Void
  consuming-site, the `if`-expr-requires-`else` semantic; finalize the
  `CaptureSource` wiring shape for the M2a closure-creation path. **Bump the stage
  gate to `Full`** + add the conformance-runner `Full` executor arm (atomically,
  per `stage.rs`). Re-verify all earlier-stage tests green at their final stages.
- **M1.11 — shadowing warning + tail marking** (+ S-45).

**[DECISION 9]** whether capture classification (cell-boxing) is in M1.10a (so
name resolution is complete in one chunk) or deferred to M1.10c (so M1.10a ships
the simpler locals+blocks+free-names and treats a cross-`fn` reference as a
provisional `Capture` placeholder). Recommending **M1.10a** — resolving an
`Ident` needs the full classification to be correct, so splitting it risks a
misresolving intermediate.

---

## 7. Decisions (status after review, 2026-07-16)

An adversarial review of these recs (2026-07-16) confirmed [1]/[3]/[4] and
`GlobalKind`/`BodyKind`/parameter-read-as-`ModuleName` SOUND, and fixed [9] +
two gaps — folded into §2/§3 above:

- **[1]** `resolutions` side table — **SOUND**, extended to cover local *decl*
  nodes (not only Idents), so `BindSlot` has a slot.
- **[2]** `Resolution` variant set — SOUND; own-frame cell-boxed access stays
  `LocalSlot`/`BlockOuter` + the `cell_boxed` flag (no fifth variant); slot-width
  conversion is overflow-**checked**.
- **[3]** `CallableInfo` grows per chunk — **SOUND** (absent > empty-default).
- **[4]** `NameRef = { name, site }` — **SOUND** (module is fixed by the executing
  `ResolvedModule`; provenance lives in the `Binding`).
- **[5]** `ExitTarget::HomeCallable` hop count? *(M1.10b)*
- **[6]** `CaptureSource` shape. *(M1.10c)*
- **[7]/[8]** S-5 / S-6 — **RESOLVED (user, 2026-07-16); App C normative** (see
  the §4/§5 banners).
- **[9]** single-pass capture — was **unsound as written**; **fixed** by per-slot
  `cell_boxed` metadata on `CallableInfo` (a late promotion flips one bit, leaving
  emitted `LocalSlot`/`BlockOuter` refs valid). Classification stays M1.10a;
  `CaptureSource` wiring M1.10c.
- **New (from review):** `CallableInfo.slot_names` (MD §17 named-locals table) and
  `cell_boxed` added; decl-node slot resolutions added.
- **S-11 RESOLVED** (user, 2026-07-17): `fn` closures **may** mutate captures,
  by-reference, shared cell (App C). Unblocks the M1.10c capture semantics.
- **S-45** already user-resolved (App C); S-39 (imported-alias assignment) stays M5.

## 8. Capture representation — RESOLVED: **B** (cell-boxed frame slots)

The M1.10a build surfaced a fork the earlier drafts glossed: a **block nested in
a closure** referencing a captured variable doesn't fit `Resolution::Capture(index)`
(a block reaches by static link, not by the closure's capture array). Reviewed
2026-07-17; **B ratified** — and it is what machine-design **already pins**: §7
"the frame slot holds the cell reference," §8 `CalObj = … + captured cells:
Vec<CellIdx>`, §10 "entering the closure **splices those cells into the new
frame**." (Option A — a separate capture array read without entering the frame —
would *contradict* §10, so A was the deviation, not B.)

**B:** a closure's captured cells occupy **cell-boxed frame slots**. Every
reference — direct or via a block static link — is `LocalSlot`/`BlockOuter` +
the `cell_boxed` flag (the machine derefs). **`Resolution::Capture` is dropped**
(the §2 sketch of it is superseded). A per-closure **capture list**
(`CaptureSource`) survives only to drive closure creation: fill each capture slot
from the enclosing env (`ParentLocal(slot)` / `ParentCapture(idx)`).

Adversarial review (all 7 axes B-SOUND; tail-reuse, multi-level threading,
mutation, GC roots, no info loss). **Pin in M1.10c:**
- Capture slots are **trailing, discovery-order** slots (forward-pass-safe — a
  late capture never renumbers already-emitted param/body slots).
- `CaptureSource` **carries its target slot** (the splice fills it).
- Multi-level: an intermediate closure that never names the var still gets a
  pass-through capture slot; the resolver threads it because all intervening `fn`
  frames are open on the frame stack when the deepest reference resolves.
- **MD §11 caveat** (tail-call frame reuse): the capture-**splice** for a closure
  tail-target must run under the §11 overwrite set — worth an explicit sentence in
  MD §11 so it isn't the next under-specified overwrite.

**Flag for M2a (not resolver work):** `Value` has **no `Cell` variant**
(`machine.rs`), yet `locals: Vec<Option<Value>>` must hold a cell reference for a
cell-boxed slot. A/B-shared for owning frames; B leans on it harder. Resolve at
M2a (`Value::Cell(CellIdx)`, a `Value|Cell` locals element, or a parallel
`cells` vec indexed by the `cell_boxed` slots) before the machine core lands.
