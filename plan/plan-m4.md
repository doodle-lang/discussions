# Working plan — M4: Language completion I (data, errors, strings)

The session-sized breakdown of milestone **M4** (`implementation.md` §5 M4,
`[L]`). M4 completes essentially the whole language surface **except modules and
protocols** (those are M5): dicts, value/ref records, structural equality, the
error system (`try`/`rescue`/`raise`), dynamic binding (`with`/`parameter`), and
full string/Unicode. Accept is **L§4–§9 + §12 green on native and wasm**, minus
the clauses that need protocol dispatch or stdlib defaults (those are carved
into M5/M9a explicitly).

> **Status:** written 2026-08-23; **revised after a 4-lens adversarial review**
> (decomposition, completeness, spec-deltas/decisions, sizing/risk). The review
> corroborated the load-bearing survey claim (the front end is complete) and
> reshaped several items — see the change log at the end.

## Where M4 lives: the machine, not the front end

A pre-plan survey + review (read-only, 2026-08-23) established the ground truth:

- **The lexer, parser, and resolver are already complete for every M4
  construct** — verified across two independent reads. Dict/record literals,
  `record`/`parameter` declarations, field access, indexing, `try`/`rescue`/
  `raise`, `with`, and interpolation all parse to real AST nodes and are
  scoped/annotated by the resolver (including the static rules, e.g. assigning to
  a `parameter` is rejected with "rebind it with `with`", `errors.rs:89-90`).
- **Almost all M4 work is in the CESK machine**, gated by **several distinct
  `unimplemented!` sites** (not just the two catch-alls): the statement catch-all
  `step.rs:300`, the expression catch-all `step.rs:429`, interpolation
  `step.rs:358` (a `StrLit` with an `Interp` part), place-chain assignment
  `control.rs:114`, and structural `==` `compare.rs:101`. The runtime value
  *tags* exist (`Value::Dict`/`Record`), but there is **no heap object,
  constructor, or op** behind them, and GC treats them as no-op stubs
  (`gc.rs:92`).
- **The unwind cleanup mechanism is UNBUILT, not dormant.** Earlier prose called
  the `WithRestore` continuation "inert since M2a.6" — that is wrong: **there is
  no `WithRestore`/`TryHandler` continuation in the code at all** (only comments
  reserving them). Every current unwind arm (`block_break`, `do_return`,
  `native_break`, `Cancel`) abandons a frame's whole cont stack with an
  unconditional `frames.pop()` and never walks it (`unwind.rs:296-302`). M4 must
  *introduce* the cleanup-cont category and *rewrite* every unwind arm to run
  cleanup as it pops — this is a bigger foundation than "activate a reserved
  hook" (drives the M4.5 split below).
- **The runtime acceptance corpus is effectively empty** — only 4 `mode: run`
  fixtures exist (82 `mode: static`). Every subsystem needs new `mode: run`
  fixtures; the many `mode: static` fixtures for records/`parameter`/
  interpolation can be promoted.

### Live bug found by the survey (M4.0 closes it first)

`[1] == [2]` **panics today**: lists became constructible at M2b.5a, but the
compound arm of structural `==` is `unimplemented!` (`compare.rs:98-102`), and
its header comment still claims compounds are "not constructible" (stale). Legal
Doodle code aborts the engine (release wasm is `panic = "abort"`). **M4.0** —
list-only cycle-safe `==`, dependency-free — closes it first, before any other
item, and builds the visited-pair memo skeleton that M4.4 extends. Tracked in
`claude-todo.md` (MAJOR).

## M4 exit criteria (accept, from `implementation.md` §5 M4)

1. **L§4–§9 + §12 conformance chapters green on native and wasm**, except
   clauses requiring protocol dispatch or stdlib defaults (carved into M5/M9a
   explicitly).
2. Composed and decomposed `"café"` are **equal** with **`length == 4`**.
3. **Family-emoji and flag-fusion** grapheme cases pass; **jamo / non-starter /
   reordering seam** cases pass (the AD4 seam-renormalization).
4. A crafted **raise inside nested `with`/`try`/blocks restores every dynamic
   binding** and shows **tail-elided frames** in the trace.
5. **Cancellation inside nested `with`/blocks restores dynamic bindings and runs
   cleanup before `Faulted(Cancelled)`**.
6. The **determinism gate is green over the full suite** (the new hashing must
   not leak nondeterminism; the double-run trace diff covers everything).

## Spec-delta obligations coming due in M4

Resolve each **in the spec** as part of the item that ships it (plan §8: edit +
App C decision-log entry + conformance/test). The ★ items need a **fresh user
ask** when reached (see Decisions).

- **★ Exceptions-as-values (E§9 / L§12; E§7.5 host-raise).** The spec **already**
  pins the load-bearing parts: exceptions **are values** (E§9), `raise value`
  raises `value` (L§12.1), `rescue e` **binds the raised value** (L§12.2), and
  the **trace is captured separately** (`Raised(exception, trace)`, E§9). What is
  *provisional* today is that the engine carries an exception as **kind + message
  only, no value slot** (`error.rs:70-77`), rendering the host value into the
  message (`HostRaised`). M4 must retire that. The genuinely **open** fork is
  narrower (see D-M4-2): the shape/provenance/forgeability of the engine's *own*
  error values. Lands with M4.5b. **Decision.**
- **S-38 (L§5.3/§4.14) — lvalue place chains + value-element copy corner.**
  Assignment targets evaluate as **places** (no intermediate value-record
  copies), so `a.inner.x = 5` mutates `a`; copies happen on binding, not on place
  navigation (currently `unimplemented!`, `control.rs:114`). **Under-specified
  corner to pin at the same time:** whether writing a value record *into* a
  container slot (`xs[0] = vrec`) copies it, and whether reading `xs[0]` returns
  a copy — value semantics implies copy-in/copy-out, but neither L§4.14 nor
  L§5.3 states it. Lands with M4.2 (the rule) + M4.3 (place navigation).
- **S-28 (L§4.13/§4.8) — dict-key hash + structural-`==` halves.** *(Was
  mis-cited as "S-24", which is the M9b REPL item — corrected.)* The **key
  hash** half + **first-key-wins key retention** (L§4.8 — which of two `==`-equal
  keys is stored) land with **M4.1**; the **structural-`==`** half (dict/record
  arms) with **M4.4**. The −0.0/NaN equality rules are already pinned (M2a.3).
- **Dict-`==` order-independence (L§4.13) — NEW small delta.** L§4.13 says dicts
  compare by "same keys and pairwise-equal values" but does **not** state whether
  equality is independent of insertion order (which is otherwise a deliberately
  observable, replay-load-bearing property, L§4.8). Add an explicit L§4.13
  sentence ("dict equality is independent of insertion order") + a fixture; do
  **not** assume it. Lands with M4.4.
- **★ S-29 (L§4.8/§15) — record-as-key model.** Pin which records are usable as
  keys. Needs a fresh ask; frame around **content mutability reachable from the
  key**, not the value/ref label (a *value* record can share a `List`/`ref`
  field, so it is as stale-prone as a `ref` record — L§4.14). Names edits to
  **L§4.8** ("records usable as keys" → the restricted rule) and **L§15 hook 2**
  (record Hashable default). Lands with M4.1. **Decision (D-M4-1).**
- **★ S-37 (L§4.12/App D.1, L§15 hook 1) — type-value spellings + interpolation
  binds Stringable directly.** Pin the built-in type-value spellings, **whether
  `Procedure` distinguishes procedures from functions** (partly a *ratify* — `is`
  already picked some behavior at M2a.5), and that **interpolation invokes the
  Stringable dispatcher directly** (immune to shadowing of `to_string`). Lands
  with M4.9. **Decision (D-M4-5).**
- **S-30 (E§4.3) — host `make_string` error model.** The host-side `make_string`
  failure mode (error return, not raise) and the general no-drive-in-progress
  host-call error model. Lands with the string boundary work (M4.7 / M4.8b).
- **★ S-10 (L§7.10) — valued `break` into a `to`-consumer.** *(Reframed: the
  loop half is ALREADY resolved — a static error, 2026-07-29.)* The **open** half
  is a valued `break` targeting a `to`-consumer (no value slot); it was reserved
  as a **fresh ask at M2a.6** and slipped — take it now. Expected shape: the S-6
  static/runtime split. Lands with M4.6. **Decision.**
- **★ R8 — mid-computation bignum-magnitude cap (Decision #9).** The deferred
  real fix for "huge `**`/`"a"*10**9` freezes the tab": bound bignum/string
  *magnitude* **during** the operation. Scheduled "with M4's string work"
  (`implementation.md`). Lands with M4.10 (or the poll-point rides M4.7). **Decision
  (D-M4-4).**

## Decisions needed (the user's calls — resolve before the dependent item)

Recommended option first; each blocks only the item(s) noted. **None block
M4.0.** D-M4-1 blocks M4.1's key model but not its scalar-key core.

1. **★ D-M4-1 · Record-as-key model (S-29). RESOLVED (user, 2026-08-23) for the
   M4.1 scope:** M4.1 ships **scalar keys only** (Int/BigInt/Float/Bool/Nil/Str/
   Bytes), self-contained and deterministic; a hand-rolled fixed-key SipHash-1-3
   through a single `hash` seam (the M5-Hashable placeholder). **Record keys defer
   to M4.4** (they need record `==`); when they land, a record is a usable key iff
   **every value transitively reachable from it is immutable** (framed by reachable
   mutability, not the value/ref label — a *value* record can share a mutable field).
   Edits due at M4.4: L§4.8, L§15 hook 2. (No "tuple" type exists.)
2. **★ D-M4-2 · Engine error-value shape (E§9).** *Blocks M4.5b.* **What is
   already pinned** (not open): exceptions are values; `rescue e` binds the
   raised value; trace is separate. **The fork:** (a) the **shape** of the
   engine's own error values — a built-in **`Error` record** (`kind`, `message`,
   …) vs. a **distinct non-forgeable error kind** vs. plain strings; (b)
   **provenance** — engine-level (must exist at M4.5b, pre-stdlib) vs. stdlib
   (M9a); (c) **forgeability** — can a kid construct a value that a
   `rescue e … if e is IndexError` check treats as an engine `IndexError`?
   **Recommend (A):** an **engine-level built-in `Error` record**, inspectable and
   (initially) forgeable — fits Doodle's no-magic stance and L§12.2's `e is …`
   idiom; revisit forgeability if spoofing matters. Edits: E§9, E§7.5, L§12.
   *(Option "always bind an Exception wrapper" is rejected — it contradicts
   L§12.2 + E§9.)* A genuine design discussion when reached.
3. **D-M4-3 · Unicode pin (D-5) + segmentation crate.** *Blocks M4.8a.* The
   `unicode-normalization` pin is UCD 17.0, but AD4 warns the crates ship skewed
   UCD versions. **Recommend:** pin all three to the **newest UCD version all
   three support** (audit `unicode-segmentation` + `unicode-ident` for a 17.0
   release) — **17.0 if they support it, else 16.0** (the original D-5 target).
   The choice changes observable grapheme counts and the replay `unicode_version`
   identity (S-41), so confirm it explicitly during the AD4 per-crate audit.
4. **★ D-M4-4 · R8 bignum/string magnitude cap.** *Blocks M4.10 only.* Land the
   mid-computation cap in M4.10 (with finer limits), or defer again?
   **Recommend:** land it — M4 touches arithmetic/string limits, the demo is
   public, and it closes Decision #9's real fix. (Marked ★ in both lists.)
5. **D-M4-5 · S-37 type-value spellings + `Procedure` distinction.** *Blocks
   M4.9's dispatcher naming, and reflection/`is`/error-message wording.* Pin the
   spellings and whether `Procedure` splits `to`/`fn`. Partly a **ratify** of the
   M2a.5 `is` behavior. **Recommend:** confirm the existing behavior + spellings;
   fresh only where reflection needs the `to`/`fn` split.

## The work items

Three loosely-independent tracks — **A: data** (M4.0–M4.4), **B: control**
(M4.5a–M4.6), **C: strings** (M4.7–M4.9) — converging at **M4.10**. Cross-track
order is flexible; the **within-track dependency edges are stated per item**
(and corrected from the first draft). Each `[S]`/`[M]`/`[L]` is a rough session
estimate.

### M4.0 — Close the `[1]==[2]` panic: list-only cycle-safe `==` `[S]` — **DONE** (doodle-rust `9ee3d9e`)

- **Goal.** Stop legal code aborting the engine; lay the equality skeleton.
- **Lands.** The **expected-fail `mode: run` fixture** (`L4.13`) reproducing
  `[1] == [2]` (first), then the **list arm** of structural `==` behind
  `compare.rs:98-102` with the **cycle-safe visited-pair memo** (deterministic —
  **not** a default-hasher `HashMap`); the stale "unreachable" comments
  (`compare.rs:19-20,96-97`) corrected.
- **Depends on.** — (dependency-free; ordered first).

### M4.1 — Deterministic hashing + heap dicts (scalar keys) `[M]` — **DONE** (doodle-rust `b9b73e8`)

- **Goal.** The first hashing infrastructure in the engine, and a working dict.
- **Lands.** A **fixed-key SipHash `BuildHasher`** (determinism-critical — no
  default hasher, no address/seed randomness; the fixed key is part of the
  determinism contract, §4.1) reached through a **single `hash` seam with a
  native placeholder** that M4.9's `hash` hook and M5's Hashable protocol later
  *fill in* (not a throwaway built-in-only path M5 must rip out); a heap
  **`DictObj`** (insertion-ordered) + `alloc_dict` + **its GC slab, mark, and
  child-scan (keys and values)** arms (`gc.rs:92`); dict-literal evaluation; get
  / set / `d[k]` (raising on a missing key); insertion-order iteration;
  **first-key-wins key retention** (L§4.8). **Scalar keys only** (D-M4-1).
  **Correctness requirement (named + tested):** because `==` is total and
  cross-kind (`Int(1)==Float(1.0)`, `Int(5)==BigInt(5)`), any two `==`-equal keys
  **must hash equal**, or `{1:"a"}[1.0]` silently breaks.
- **Spec-delta.** S-28 (dict-key hash half + first-key-wins); S-29 (per D-M4-1).
- **Tests.** `mode: run` dict fixtures; a **determinism unit test** (iteration is
  construction-order, hash-independent); the **cross-kind-equal-keys-hash-equal**
  test; the GC-stress gate over dicts.
- **Depends on.** D-M4-1 (key set).

### M4.2 — Records: heap repr, constructor, field read, `is`, copy-on-bind `[M]`

- **Goal.** Records as first-class values.
- **Lands.** A heap **`RecObj`** with the value/ref header (L§4.14) + **its GC
  slab, mark, and field-scan** arms; a **record type-value + constructor**
  (`Point(x: 3, y: 4)`); **field read** (`p.x`); **record-nominal `is`**
  (`x is Point` — extend `types::is_op`, which assumes a builtin type today,
  `types.rs:85-95`); **copy-on-bind** for value records (copy on binding/passing;
  `ref` records share) — including the **copy-on-store/copy-on-read** rule for a
  value record placed into / read from a container slot (S-38 corner).
- **Spec-delta.** S-38 (the value-element copy corner, with M4.3).
- **Design refs.** L§4.14, L§9, L§6.5 (record `is`); `value.rs:68`.
- **Tests.** `mode: run` promotions of the `L9.1`/`L4.14` static fixtures
  (copy-on-bind vs. ref-share); field read; constructor arity/errors;
  `x is Point` + cross-type-false; `xs[0]=vrec; vrec.x=9` must not change `xs[0]`.
- **Risk.** copy-on-bind × place-chain aliasing (with M4.3) — name it here.
- **Depends on.** —

### M4.3 — Place chains: lvalue assignment (S-38) `[M]`

- **Goal.** `a.b.c = x` and `d[k] = v` mutate in place, no intermediate copies.
- **Lands.** Place navigation for the LHS (`Ident | Field | Index`, already
  parsed) replacing `unimplemented!("… place is M4 (S-38)")` (`control.rs:113`);
  copies happen on binding, not on place navigation (the S-38 invariant),
  reconciled with M4.2's copy-on-store/read rule.
- **Spec-delta.** S-38.
- **Tests.** `a.inner.x = 5` mutates `a` (value-record `inner`, no mid-chain
  copy); `d[k] = v`; a value record copied on bind then mutated leaves the
  original unchanged; a `ref` record does not; `xs[0].x = 9` mutates vs.
  `let y = xs[0]; y.x = 9` does not.
- **Risk.** the aliasing edge cases above are the error-prone core (re-tagged
  `[S]→[M]` after review).
- **Depends on.** M4.1 (dict places), M4.2 (record places). Lists exist.

### M4.4 — Structural `==` (dict/record arms) + compound dict keys `[S]`

- **Goal.** Extend M4.0's equality to dicts/records; enable record keys.
- **Lands.** The dict and record arms of the cycle-safe walk (extending M4.0's
  memo); **dict `==` order-independent** per the new L§4.13 delta; record `==`;
  and **compound (record) dict-key support** (rides here since it needs record
  `==` + M4.2).
- **Spec-delta.** S-28 (structural-`==` half); dict-`==` order-independence; S-29
  (record keys land here).
- **Tests.** `mode: run`: nested/cyclic structures; `{a:1,b:2} == {b:2,a:1}`;
  record `==`; a record used as a dict key.
- **Depends on.** M4.0 (the memo), M4.1 (dict), M4.2 (record).

### M4.5a — Unwind-cleanup foundation (shared) `[M]`

- **Goal.** The one mechanism `with`, `try`, and cancellation-cleanup all need.
- **Lands.** Introduce the **cleanup-cont category** (`WithRestore` +
  `TryHandler` conts — *new*, not reserved); **rewrite every unwind arm**
  (`block_break`, `do_return`, `native_break`, `Cancel`) to **walk each abandoned
  frame's conts and execute cleanup as it pops** (replacing the unconditional
  `frames.pop()`); add an **`Unwind::Raise`** variant carrying the exception, so
  the raise path unwinds through the **frame channel** (not `Err(Raise)` straight
  to the boundary, `drive.rs:354`) and therefore also runs cleanup.
- **Design refs.** MD §12; `unwind.rs`, `cont.rs`, `step.rs`, `drive.rs`.
- **Tests.** unit: each exit tier (`break`/`continue`/`return`/`cancel`/raise)
  runs a registered cleanup cont as it unwinds.
- **Depends on.** — (independent of Track A and of D-M4-2; unblocks M4.6).

### M4.5b — `try`/`rescue`/`raise` + exceptions-as-values `[L]`

- **Goal.** The error control flow and the exception value model.
- **Lands.** Handler binding via the `TryHandler` cont (M4.5a); **user `raise`**
  (construct + throw) and **bare re-raise** (in-flight-exception tracking);
  **exceptions as values** — `Exception` gains a value slot, the E§7.5 host-raise
  representation settled, `Outcome::Raised` carries the value, `rescue e` binds
  it (per D-M4-2); `try` as an expression (L§6.9).
- **Spec-delta.** ★ exceptions-as-values (E§9/§7.5/L§12, per D-M4-2).
- **Tests.** `mode: run` L§12 fixtures: raise → rescue binds the value; `e is …`;
  re-raise; raise crossing block/`to`/`fn` frames; uncaught reaches the boundary.
- **Depends on.** M4.5a; D-M4-2; **and — under the recommended engine-error-record
  answer — M4.2** (engine errors materialize as record values).

### M4.5c — Trace capture `[S]`

- **Goal.** Traces that show where an error originated.
- **Lands.** extend `Trace` (today `raised_at` only, `error.rs:84-88`) with
  **live frames + bounded tail-elided history**; wire `observe::stack_walk`
  (`observe.rs:79`) into the raise; keep it deterministic.
- **Tests.** a raise deep in tail-recursion shows tail-elided frames (accept #4).
- **Depends on.** M4.5b.

### M4.6 — `with`/`parameter` runtime + cancellation cleanup `[M]`

- **Goal.** Dynamic binding with airtight restoration.
- **Lands.** **Parameter cells** + a **dynamic-binding stack** on the machine;
  `with X = v do … end` establish/restore; **restoration on EVERY exit path**
  (normal, `break`/`continue`/`return`, raise, cancel) via M4.5a's cleanup conts
  (the `WithRestore` cont); per-frame exposure; **cancellation inside nested
  `with`/blocks restores bindings and runs cleanup before `Faulted(Cancelled)`**
  (accept #5, which needs only M4.5a + the existing `Unwind::Cancel`).
- **Spec-delta.** ★ S-10 (valued `break` into a `to`-consumer).
- **Design refs.** L§7.8/§7.10, MD §12.
- **Tests.** `mode: run`: nested `with` restored on each exit tier; **a cancel
  mid-`with` restores + runs cleanup before `Faulted(Cancelled)`** (accept #5);
  a raise through nested `with` restores all bindings (accept #4 — needs M4.5b).
- **Depends on.** **M4.5a** (not M4.5b). Accept #4's raise-through-`with`
  additionally needs M4.5b.

### M4.7 — String concat + repetition + AD4 seam-renormalization `[M]`

- **Goal.** `+` and `*` on strings, done right at the seam.
- **Lands.** String **`+`** (concat) and **`*`** (repetition, String × Int —
  L§4.4/§6.5), both routing off `arith`'s number-only path; the **AD4
  seam-renormalizing pass**, **exported as a reusable routine** (M4.9 joins
  interpolation parts through it) rather than buried in binary `+`; the
  string-churn heap/fragmentation benchmark joins the suite. The R8 interior
  poll-point for a large `*` count ties into M4.10.
- **Spec-delta.** S-30 (host `make_string` error model).
- **Design refs.** AD4; L§4.4; MD §5 (the seam pass); `heap/objects.rs:13`.
- **Tests.** `mode: run`: composed+decomposed concat renormalizes at the seam;
  `s * 3` incl. a seam case; the jamo/non-starter/reordering fixtures.
- **Depends on.** —

### M4.8a — Graphemes + runtime indexing `[M]`

- **Goal.** L§4.4 grapheme semantics.
- **Lands.** the **`unicode-segmentation`** dependency (pinned, D-M4-3); the lazy
  **grapheme memo** on `StrObj` — honoring **MD §5's invariant that it is a pure
  cache EXCLUDED from `bytes_allocated`** (so it never shifts when
  `LimitExceeded(heap)` fires — a determinism hazard) and built consistently
  across all construction paths; **grapheme `length` / index / iteration**;
  runtime `s[i]` (the `Node::Index` eval arm, shared with list/dict indexing).
- **Spec-delta.** D-M4-3 (Unicode pin + segmentation crate).
- **Design refs.** L§4.4, MD §5.
- **Tests.** accept #2/#3 (`"café"` length 4; family-emoji, flags); grapheme
  index/iterate; the UCD grapheme vectors (feed M4.10).
- **Depends on.** M4.7 (soft — shares `StrObj` growth); D-M4-3.

### M4.8b — Bytes ↔ string bridging `[S]`

- **Goal.** String/bytes interop.
- **Lands.** encode/decode with the **round-trip law**; O(1) `Bytes` indexing
  (the `Bytes` type already exists); the S-30 error model at this boundary.
- **Spec-delta.** S-30 (with M4.7).
- **Design refs.** L§3.6.5, L§4.4; AD4.
- **Tests.** bytes↔string round-trip; O(1) index; invalid-UTF-8 decode behavior.
- **Depends on.** — (independent of M4.8a; could fold into M4.7's boundary work).

### M4.9 — Interpolation + the Stringable dispatcher (native placeholder) `[M]`

- **Goal.** `"{expr}"` evaluates; the pinned `to_string` seam.
- **Lands.** the **Stringable dispatcher** invoked **directly** by interpolation
  (S-37 — immune to shadowing of `to_string`), running against **native
  placeholder `to_string`/`hash` hooks** (replaced by protocol dispatch at M5,
  stdlib defaults at M9a); interpolation evaluation (`step.rs:358`) as an
  **N-ary seam join reusing M4.7's exported seam routine** (so a multi-part
  interpolation is provably equal to a single NFC pass); the provisional
  `render()` (`builtins.rs:197`) retired into the dispatcher.
- **Spec-delta.** ★ S-37 (per D-M4-5).
- **Design refs.** L§15 (Stringable hook), L§3.6.3.
- **Tests.** `mode: run` interpolation (promote `L3.6.3`); immune to a local
  `to_string`; compound placeholder rendering.
- **Depends on.** M4.7 (concat/seam) + the value types (M4.1/M4.2) for compound
  rendering. **Not** M4.8.

### M4.10 — UCD vectors, R8 cap, M4 exit review `[M]`

- **Goal.** The milestone gate.
- **Lands.** **UCD conformance vectors** in CI (normalization + segmentation,
  incl. the AD4 seam cases); the **full-suite determinism gate** green; **★ the
  R8 mid-computation bignum/string magnitude cap** (per D-M4-4); an
  accept-criteria walk (#1–#6) and a **multi-lens review** (hashing determinism +
  cross-kind hash/`==` consistency, the unwind foundation across try/with/cancel,
  place-chain aliasing, the AD4 seam, exceptions-as-values).
- **Spec-delta.** ★ R8 (per D-M4-4); close all M4 App C entries (S-28/S-29/S-30/
  S-37/S-38/S-10/exceptions-as-values/dict-`==`).
- **Depends on.** M4.0–M4.9 (and the D-M4-4 call).

## Notes on ordering and risk

- **Critical path.** Track A: **M4.0 → M4.1 → M4.2 → M4.4**, with **M4.3** after
  M4.2 (needs M4.1+M4.2). Track B: **M4.5a → {M4.5b → M4.5c, M4.6}** — M4.6 hangs
  off **M4.5a**, not the try feature. Track C: **M4.7 → M4.8a** (soft, shared
  `StrObj`) and **M4.7 → M4.9** (hard); **M4.8a, M4.8b, M4.9 are independent
  branches off M4.7**. All converge at **M4.10**. A sensible serial order:
  **4.0, 4.1, 4.2, 4.4, 4.3, 4.5a, 4.6, 4.5b, 4.5c, 4.7, 4.8a, 4.9, 4.8b, 4.10.**
- **Risk peak — the unwind foundation (M4.5a).** It is **unbuilt**, not dormant:
  introduce the cleanup-cont category and rewrite every unwind arm, and move the
  raise path onto the frame-unwind channel. `try`, `with`, and the S-23 cancel
  cleanup all ride it; get it right once and M4.6 + accept #5 fall out; get it
  wrong and every non-local exit corrupts. Full review treatment.
- **Determinism — the first hasher (M4.1).** The engine's first hasher, on a
  Doodle-observable path. Two hard constraints: fixed-key + address-free (order
  never leaks), **and cross-kind hash/`==` consistency** (`==`-equal Int/BigInt/
  Float keys hash equal). Both get named tests. The M4.4 visited-pair memo and
  any internal map must also avoid the default hasher.
- **The hash seam is locked in early.** M4.1's dict-key hash must go through the
  same `hash`-dispatch seam M4.9's hook and M5's protocol later fill in — because
  M5's dispatched `hash` **can call user code and suspend**. Design the M4.1 seam
  to tolerate that (or accept a documented `DictObj`/rehash rework at M5).
- **AD4 seam correctness (M4.7).** The milestone's named Unicode hazard —
  renormalize the join, not the whole string; export it as a routine so `+`, `*`,
  and interpolation share one proven path; prove it with the jamo/non-starter/
  reordering vectors.
- **Copy-on-bind × place-chain aliasing (M4.2/M4.3).** Value-record copy timing
  interacts directly with in-place place-chain writes and the container
  copy-in/copy-out corner — the subtle-bug epicenter; fixtures in both items.
- **Exceptions-as-values (M4.5b) is a fresh design decision (D-M4-2)** — but a
  **narrow** one: what `rescue` binds is already normative; only the engine's own
  error-value shape/provenance/forgeability is open. Resolve it in E/L before
  M4.5b ships.
- **Scope realism.** M4 is the largest language milestone; M4.5b and M4.7/M4.8a
  are the likeliest to each need 2 sessions. Protocol dispatch and stdlib
  `to_string`/`hash` defaults are **M5/M9a** and are carved out of M4 acceptance
  explicitly (the M4.9 dispatcher runs against native placeholders).

## Change log (from the first draft, after the 4-lens review)

- **Added M4.0** (list-only `==`, dependency-free) to actually close the
  `[1]==[2]` panic first, instead of burying it in M4.4.
- **Split M4.5 → M4.5a/b/c** (unwind foundation / try+exceptions / trace) and
  **re-pointed M4.6 at M4.5a**, decoupling `with` from the try feature and the
  ★ D-M4-2 decision. Corrected "`WithRestore` inert since M2a.6" → **unbuilt**.
- **Split M4.8 → M4.8a (graphemes) / M4.8b (bytes bridging).**
- **Added string `*` repetition** to M4.7 and **record-nominal `is`** to M4.2
  (both were dropped).
- **Corrected S-24 → S-28** throughout (S-24 is the M9b REPL item).
- **Reframed D-M4-2** around the genuinely-open fork (engine error-value
  shape/provenance/forgeability) and dropped the spec-contradicting "Exception
  wrapper" option; **reframed D-M4-1** around reachable mutability (dropped the
  nonexistent "tuple"); made **D-M4-3** conditional (17.0-if-supported-else-16.0);
  **added D-M4-5** (S-37); made **R8's ★** consistent.
- **Named** the cross-kind hash/`==` consistency requirement (M4.1), the hash-seam
  lock-in, the value-element copy corner (S-38), dict-`==` order-independence as a
  real delta, GC trace arms in Lands (M4.1/M4.2), the grapheme-memo
  excluded-from-accounting invariant (M4.8a), and **S-10's reserved fresh ask**.
- **Corrected** the Track-C shape (4.7 → {4.8, 4.9}, not linear) and the
  "two catch-all panics" headline (several `unimplemented!` sites).
