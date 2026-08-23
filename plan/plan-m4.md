# Working plan — M4: Language completion I (data, errors, strings)

The session-sized breakdown of milestone **M4** (`implementation.md` §5 M4,
`[L]`). M4 completes essentially the whole language surface **except modules and
protocols** (those are M5): dicts, value/ref records, structural equality, the
error system (`try`/`rescue`/`raise`), dynamic binding (`with`/`parameter`), and
full string/Unicode. Accept is **L§4–§9 + §12 green on native and wasm**, minus
the clauses that need protocol dispatch or stdlib defaults (those are carved
into M5/M9a explicitly).

## Where M4 lives: the machine, not the front end

A pre-plan survey (read-only, 2026-08-23) established the ground truth:

- **The lexer, parser, and resolver are already complete for every M4
  construct.** Dict/record literals, `record`/`parameter` declarations, field
  access, indexing, `try`/`rescue`/`raise`, `with`, and interpolation all parse
  to real AST nodes and are scoped/annotated by the resolver (including the
  static rules, e.g. "rebind a `parameter` with `with`").
- **Almost all M4 work is in the CESK machine**, behind two catch-all panics:
  `dispatch_stmt` → `machine/step.rs:300` ("statement not yet in the machine")
  and `eval` → `machine/step.rs:429` ("expression not yet in the machine"). The
  M4 nodes fall through to these. The runtime value *tags* exist
  (`Value::Dict`/`Record`), but there is **no heap object, constructor, or op**
  behind them.
- **The runtime acceptance corpus is effectively empty** — only 4 `mode: run`
  fixtures exist (all M2a-era). Every subsystem needs new `mode: run` fixtures;
  many `mode: static` (parse-level) fixtures already exist for records,
  `parameter`, and interpolation and can be promoted.

### Live bug found by the survey (fix early in M4)

`[1] == [2]` **panics today**: lists became constructible at M2b.5a, but the
compound arm of structural `==` is `unimplemented!` (`compare.rs:98-102`), and
its header comment still claims compounds are "not constructible" (stale). This
is legal Doodle code aborting the engine. The first implementation step of M4 is
an **expected-fail conformance fixture** reproducing it; it closes as part of
M4.4 (structural `==`), whose list arm is the only currently-reachable case.
Tracked in `claude-todo.md`.

## M4 exit criteria (accept, from `implementation.md` §5 M4)

1. **L§4–§9 + §12 conformance chapters green on native and wasm**, except
   clauses requiring protocol dispatch or stdlib defaults (carved into M5/M9a
   acceptance explicitly).
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

- **★ Exceptions-as-values (E§9 / L§12; E§7.5 host-raise).** `Exception` today
  is **kind + message only, no value slot** (`error.rs:70-77`); the host-raise
  path renders the value into the message as a placeholder (`HostRaised`). M4
  must pin **what a raised exception carries** and **what `rescue name` binds**,
  and settle the E§7.5 host-raise representation. Lands with M4.5. **Decision.**
- **S-38 (L§5.3/§4.14) — lvalue place chains.** Assignment targets evaluate as
  **places** (no intermediate value-record copies), so `a.inner.x = 5` mutates
  `a`; copies happen on binding, not on place navigation. Currently
  `unimplemented!` (`control.rs:113-115`). Lands with M4.3.
- **S-24 (L§4.13/§4.8) — the dict-key hash + structural-`==` halves.** The
  fixed-seed dict-key hashing lands with M4.1; the structural-`==` half with
  M4.4 (first-key-wins dict `==`, −0.0/NaN rules already pinned at M2a.3).
- **★ S-29 (L§4.8/§15) — mutable records as dict keys.** Pin the behavior:
  allow, with documented stale-hash reality, **or** restrict default Hashable to
  immutable content. Needed for M4.1's key model. **Decision.**
- **S-37 (L§4.12/App D.1, L§15 hook 1) — type-value spellings + interpolation
  binds Stringable directly.** Pin the built-in type-value spellings, whether
  `Procedure` distinguishes procedures from functions, and that **interpolation
  invokes the Stringable dispatcher directly** (immune to lexical shadowing of
  `to_string`). Lands with M4.9.
- **S-30 (E§4.3) — host `make_string` error model.** The host-side
  `make_string` failure mode (error return, not raise) and the general
  no-drive-in-progress host-call error model. Lands with M4.7/M4.8 (the string
  boundary work).
- **S-10 (L§7.10) — `break`-with-value where the consumer has no value slot.**
  Specify discard vs. error. Lands with M4.6 (the exit/unwind work).
- **★ R8 — mid-computation bignum-magnitude cap (Decision #9).** The deferred
  real fix for the "huge `**` freezes the tab" hazard: bound bignum *magnitude*
  **during** the operation. `implementation.md` scheduled it "likely with M4's
  finer limits." **Decision** whether it lands in M4.10 or defers again.

## Decisions needed (the user's calls — resolve before the dependent item)

Recommended option first; each blocks only the item(s) noted. None block M4.1's
start except D-M4-1.

1. **★ D-M4-1 · Dict key model (S-29).** *Blocks M4.1.* Which keys are Hashable,
   and how are mutable-content records handled as keys?
   **Recommend:** ship M4.1 with **scalar + immutable-content keys** (Int/BigInt/
   Float/Bool/Nil/Str/Bytes, and value-records/tuples of those); a mutable
   (`ref`) record used as a key is either rejected as not-Hashable or keyed by
   identity — pin which. This keeps the first hashing implementation
   deterministic and avoids stale-hash foot-guns; the full Hashable-protocol
   story is M5/M9a.
2. **★ D-M4-2 · Exceptions-as-values shape (E§9).** *Blocks M4.5.* What does
   `raise X` carry and what does `rescue e` bind? Options: **(A)** `raise` any
   value; engine-generated errors raise a **built-in `Error` record**
   (`kind`, `message`, …), and `rescue e` binds the raised value. **(B)** `rescue
   e` always binds an **Exception object** wrapping kind + message + value +
   trace. **Recommend (A)** — "everything is a value" fits Doodle's no-magic
   stance and L§12's direction; the engine's own errors are just records a kid
   can inspect and `raise` themselves. Needs a real discussion.
3. **D-M4-3 · Unicode pin (D-5) + segmentation crate.** *Blocks M4.8.* The
   `unicode-normalization` pin is now **Unicode 17.0.0** (survey), not the 16.0
   D-5 named. **Recommend:** pin v0.1 to **Unicode 17.0** and add
   **`unicode-segmentation`** at a version whose UCD matches (the AD4 per-crate
   audit), cross-checked at build like the existing `unicode-ident` check.
4. **D-M4-4 · R8 bignum cap scheduling.** *Blocks M4.10 only.* Land the
   mid-computation magnitude cap in M4.10 (with finer limits), or defer again?
   **Recommend:** land it in M4.10 — M4 already touches arithmetic limits, the
   demo is public, and it closes Decision #9's real fix.

## The work items

Three loosely-independent tracks — **A: data** (M4.1–M4.4), **B: control**
(M4.5–M4.6), **C: strings** (M4.7–M4.9) — converging at **M4.10**. Within a
track the order is a hard dependency chain; across tracks the order is flexible
(B and C can interleave with A). Each `[S]`/`[M]` is a rough session estimate.

### M4.1 — Deterministic hashing + heap dicts `[M]`

- **Goal.** The first hashing infrastructure in the engine, and a working dict.
- **Lands.** A **fixed-key SipHash `BuildHasher`** (determinism-critical — no
  default hasher, no address/seed randomness; the fixed key is part of the
  determinism contract, §4.1); a heap **`DictObj`** (insertion-ordered) +
  `alloc_dict`; dict-literal evaluation (`{k: v}`, computed keys); get / set /
  `d[k]` indexing (raising on a missing key per L); insertion-order iteration.
  Dict `==` is deferred to M4.4. Key hashing covers the D-M4-1 key set.
- **Spec-delta.** S-24 (dict-key hash half); S-29 (per D-M4-1).
- **Design refs.** L§4.7/§4.8; §4.1 (determinism); the module namespace's
  hashing-free linear scan (`control.rs`) stays as-is.
- **Tests.** `mode: run` dict fixtures (literal, get/set, missing-key raise,
  insertion-order iteration); a **determinism unit test** that dict iteration
  order is construction-order and hash-independent; the GC-stress gate over dicts.
- **Depends on.** D-M4-1.

### M4.2 — Records: heap repr, constructor, field read, copy-on-bind `[M]`

- **Goal.** Records as first-class values.
- **Lands.** A heap **`RecObj`** with the value/ref header (L§4.14); a **record
  type-value + constructor** (`Point(x: 3, y: 4)`) replacing the `TypeObj`
  builtin-only stub; **field read** (`p.x`); **copy-on-bind** for value records
  (a value record copies when bound/passed; a `ref` record shares).
- **Design refs.** L§4.14, L§9; `value.rs:68` (the header contract).
- **Tests.** `mode: run` promotions of the existing `L9.1`/`L4.14` static
  fixtures (copy-on-bind vs. ref-share); field read; constructor arity/errors.
- **Depends on.** —

### M4.3 — Place chains: lvalue assignment (S-38) `[S]`

- **Goal.** `a.b.c = x` and `d[k] = v` mutate in place, no intermediate copies.
- **Lands.** Place navigation for the LHS (`Ident | Field | Index`, already
  parsed) replacing `unimplemented!("… place is M4 (S-38)")` (`control.rs:113`);
  copies happen on binding, not on place navigation (the S-38 invariant).
- **Spec-delta.** S-38.
- **Tests.** `mode: run`: `a.inner.x = 5` mutates `a`; `d[k] = v`; a value-record
  copied on bind then mutated does not affect the original; a `ref` record does.
- **Depends on.** M4.1 (dict places), M4.2 (record places). Lists exist.

### M4.4 — Structural, cycle-safe `==` `[S]`

- **Goal.** Total structural equality over compounds; close the live panic.
- **Lands.** The recursive **cycle-safe** structural walk (list, dict, record)
  behind `compare.rs:98-102`; dict `==` per L§4.13 (first-key-wins semantics
  already pinned); list `==`; record `==`; the visited-pair memo for cycles;
  the stale "unreachable" comments (`compare.rs:19-20,96-97`) corrected.
- **Spec-delta.** S-24 (structural-`==` half).
- **Tests.** the expected-fail `[1]==[2]` fixture flips to pass; nested/cyclic
  structures; dict order-independence per L§4.13; cross-kind inequality.
- **Depends on.** M4.1, M4.2 (for dict/record ==). The list arm can land first
  and immediately closes the panic.

### M4.5 — Unwind foundation + `try`/`rescue`/`raise` + exceptions-as-values `[L]`

- **Goal.** The error system, and the unwind-continuation mechanism it shares
  with `with` and cancellation cleanup.
- **Lands.** The **handler-search unwind** (a `TryHandler` continuation the
  unwinder recognizes as it pops — the mechanism `unwind.rs:10-11` reserves for
  M4); **user `raise`** (construct + throw), **bare re-raise**; **exceptions as
  values** (`Exception` gains a value slot per D-M4-2; the E§7.5 host-raise
  representation settled); **trace capture at raise** (live frames + bounded
  tail-elided history, wiring the existing `observe::stack_walk` into the raise).
- **Spec-delta.** ★ exceptions-as-values (E§9/§7.5, per D-M4-2); S-10 note.
- **Design refs.** L§12, L§7.9; E§9; `error.rs`, `unwind.rs`, `cont.rs`.
- **Tests.** `mode: run` L§12 fixtures: raise → rescue binds the value; re-raise;
  raise crossing block/`to`/`fn` frames; trace shows tail-elided frames;
  uncaught reaches the boundary unchanged.
- **Depends on.** D-M4-2. (Independent of Track A; can start in parallel.)

### M4.6 — `with`/`parameter` runtime + cancellation cleanup `[M]`

- **Goal.** Dynamic binding with airtight restoration.
- **Lands.** **Parameter cells** + a **dynamic-binding stack** on the machine;
  `with X = v do … end` establish/restore; **restoration on EVERY exit path**
  (normal, `break`/`continue`/`return`, raise, cancel) via the M4.5 unwind
  foundation (the `WithRestore` continuation, inert since M2a.6); per-frame
  exposure; **cancellation inside nested `with`/blocks restores bindings and runs
  cleanup before `Faulted(Cancelled)`** (accept #5).
- **Spec-delta.** S-10 (with the exit work).
- **Design refs.** L§7.8, MD §12; `unwind.rs` (`WithRestore`).
- **Tests.** `mode: run`: nested `with` restored on each exit tier; a raise
  through nested `with` restores all bindings; **a cancel mid-`with` restores +
  runs cleanup before `Faulted(Cancelled)`** (drives the S-23 machinery).
- **Depends on.** M4.5 (the unwind foundation).

### M4.7 — String concat + AD4 seam-renormalization `[M]`

- **Goal.** `+` on strings, done right at the seam.
- **Lands.** String `+` (routing off `arith`'s number-only path); the **AD4
  seam-renormalizing concat** (renormalize only across the join, not the whole
  string — jamo/non-starter/reordering-indicator cases); the string-churn
  heap/fragmentation benchmark joins the suite.
- **Spec-delta.** S-30 (host `make_string` error model, with the boundary work).
- **Design refs.** AD4; L§4.3; `heap/objects.rs:13` (the reserved seam pass).
- **Tests.** `mode: run`: composed+decomposed concat renormalizes at the seam;
  the jamo/non-starter/reordering fixtures; associativity where NFC allows.
- **Depends on.** —

### M4.8 — Graphemes, runtime indexing, bytes bridging `[M]`

- **Goal.** L§4.4 grapheme semantics and string/bytes interop.
- **Lands.** the **`unicode-segmentation`** dependency (pinned, D-M4-3); a lazy
  grapheme memo on `StrObj`; **grapheme `length` / index / iteration**; runtime
  `s[i]` (the `Node::Index` eval arm); **Bytes↔string bridging** (encode/decode
  with the round-trip law, O(1) `Bytes` indexing).
- **Spec-delta.** D-M4-3 (Unicode 17 pin + segmentation crate); S-30.
- **Design refs.** L§4.4, L§3.6.5; AD4.
- **Tests.** accept #2/#3 fixtures (`"café"` length 4; family-emoji, flags);
  grapheme index/iterate; bytes round-trip; UCD grapheme vectors (feeds M4.10).
- **Depends on.** M4.7 (shares the string-object growth); D-M4-3.

### M4.9 — Interpolation + the Stringable dispatcher (native placeholder) `[M]`

- **Goal.** `"{expr}"` evaluates; the pinned `to_string` seam.
- **Lands.** the **Stringable dispatcher** invoked **directly** by interpolation
  (S-37 — immune to shadowing of `to_string`), running against **native
  placeholder `to_string`/`hash` hooks** (replaced by protocol dispatch at M5,
  stdlib defaults at M9a); interpolation evaluation (`step.rs:356-359`); the
  provisional `render()` retired into the dispatcher.
- **Spec-delta.** ★ S-37 (type-value spellings, `Procedure` distinction,
  interpolation-binds-Stringable-directly).
- **Design refs.** L§15 (Stringable hook), L§3.6.3; `builtins.rs:197` (render).
- **Tests.** `mode: run` interpolation (promote the `L3.6.3` fixtures);
  interpolation immune to a local `to_string`; compound placeholder rendering.
- **Depends on.** M4.7 (concat), and the value types (M4.1/M4.2) for compound
  rendering.

### M4.10 — UCD vectors, R8 cap, M4 exit review `[M]`

- **Goal.** The milestone gate.
- **Lands.** **UCD conformance vectors** in CI (normalization + segmentation,
  incl. the AD4 seam cases); the **full-suite determinism gate** green; **★ the
  R8 mid-computation bignum-magnitude cap** (per D-M4-4); an accept-criteria
  walk (#1–#6) and a **multi-lens review** (hashing determinism, the unwind
  foundation across try/with/cancel, place-chain aliasing, the AD4 seam,
  exceptions-as-values).
- **Spec-delta.** ★ R8 (per D-M4-4); close all M4 App C entries.
- **Depends on.** M4.1–M4.9 (and the D-M4-4 call).

## Notes on ordering and risk

- **Critical path.** Track A: **M4.1 → M4.2 → M4.3 → M4.4** (each needs the prior
  heap object). Track B: **M4.5 → M4.6** (the unwind foundation is shared). Track
  C: **M4.7 → M4.8 → M4.9**. All three converge at **M4.10**. B and C can
  interleave with A; a sensible serial order is 4.1, 4.2, 4.4 (close the panic),
  4.3, 4.5, 4.6, 4.7, 4.8, 4.9, 4.10.
- **Risk peak — the unwind foundation (M4.5).** `try`/`with`/cancellation
  cleanup all ride the same "continuations the unwinder executes as it pops"
  mechanism, currently inert. Getting it right once (M4.5) makes M4.6 and the
  S-23 cleanup path fall out; getting it wrong corrupts every non-local exit.
  Give it the full review treatment.
- **Determinism — the new hashing (M4.1).** This is the **first hasher** in the
  engine, on a Doodle-observable path (dict iteration is insertion-ordered, but
  the hasher must still be fixed-key and address-free so bucketing never leaks).
  A determinism unit test lands with M4.1; the double-run gate covers the rest.
- **AD4 seam correctness (M4.7).** Seam-renormalization is the milestone's named
  Unicode hazard — renormalize the join, not the whole string, and prove it with
  the jamo/non-starter/reordering vectors. Wrong seam handling breaks accept #3.
- **Exceptions-as-values (M4.5) is a fresh design decision (D-M4-2),** not a
  re-confirm — it sets what `rescue` binds for the life of the language. Resolve
  it in E before M4.5 ships.
- **Scope realism.** M4 is the largest language milestone; several items (M4.5
  especially, likely M4.1/M4.8) will each split into 2–3 sessions when reached —
  sequencing, not scope-cut. Protocol dispatch and stdlib `to_string`/`hash`
  defaults are **M5/M9a** and are carved out of M4 acceptance explicitly (the
  M4.9 dispatcher runs against native placeholders).
