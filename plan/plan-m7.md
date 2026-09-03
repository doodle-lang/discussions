# Plan — M7: C ABI + native CLI host `[L]`

The working plan for milestone **M7** (implementation plan §5, the M7 paragraph):
the engine's **C ABI** (`doodle-capi`) with a committed, generated `doodle.h`,
and the native **`doodle` CLI** — the second and third embedding surfaces beside
the browser (wasm) one, certified to run the conformance suite with **traces
identical across all three surfaces**.

Status: **RATIFIED (user, 2026-09-02)** — D-M7-1 resolved earlier
(`c377b32`); D-M7-2..11 ratified with two adjustments (D-M7-8 immutable-only
defaults; D-M7-10 dry-run-ready, publication behind D-8) and two riders
(finalizer-as-determinism-guard; platform-neutral conventions). Execution may
begin at M7.0. Chunk sizes per implementation-plan Appendix A.

> **Milestone size — re-tagged `[M]` → `[L]` (review finding).** The milestone
> contains a genuine `[L]` chunk and, honestly counted (the certification
> substrate below + threads + two sanitizer toolchains + Miri bring-up +
> three-surface parity), the serial critical path is `[L]`. (House-style note:
> M6 was likewise a milestone-`[M]` containing an `[L]` chunk — a milestone tag
> reads as the two-engineer critical path; flagged for Appendix A to define.)

## Revisions from the 4-lens review (2026-09-02)

Four read-only lenses (FFI/`unsafe` soundness, ABI stability, spec/determinism,
decomposition/scope) stress-tested the draft. The plan's architecture held —
M7 is over an engine that already enforces the hard invariants — but the review
surfaced **one determinism Critical, four soundness Criticals, two ABI-freeze
Criticals, and two decomposition Criticals**, plus new decisions the freeze
locks in. Folded below; the biggest changes:

1. **The freeze needs real ABI discipline** — dedicated `#[repr(C)]` types owned
   by `doodle-capi` (not `doodle-core`'s default-repr enums), a growth mechanism
   on every by-value struct, symbol prefixing, and an ABI version distinct from
   the engine version. Frozen as **conventions in M7.1** before M7.2/M7.3 copy
   them. (New M7.1 scope; D-M7-3 rewritten; new D-M7-7.)
2. **Strings/handles cross by copy, not by aliasing** — the wasm boundary is
   safe because it *copies out*; the drafted "mirror the `&[u8]` contract" would
   hand out interior pointers. (D-M7-6, new.)
3. **`Instance` is `!Send` today** (the `Finalizer` type lacks `+ Send`) — an
   M7.0 obligation, not a one-line claim. (D-M7-5, M7.0.)
4. **`doodle_call_block` is control-inversion, not marshalling** — the block
   handle must be the callback's `ctx` pointer, never the instance, or two live
   `&mut Instance` = UB. (D-M7-2, M7.2.)
5. **Panic-across-FFI** must be caught at every boundary; the finalizer
   trampoline is abort-on-unwind. (M7.1.)
6. **S-19 (synchronous-FF determinism) is the load-bearing contract M7 first
   ships** — `time`/`random` must be *capabilities*, not sync FFs, or replay
   leaks off-record. Added to the spec-delta obligations. (Critical; D-M7-4.)
7. **The cross-surface trace was never defined, and the C-conformance mechanism
   was a parenthetical** — both promoted to a defined schema + a real chunk
   (M7.5). The drive-script format needs capability-resolution steps (deferred
   here from M6.8) and a portable test-capability registry. (New M7.5.)
8. **Three silent carve-outs → flagged decisions**: Linux-only C certification
   (D-M7-9), which crates/artifacts publish (D-M7-10), and the S-42 default
   *evaluation timing* vs L§8.3 (D-M7-8, a semantics question).

## The one fact that dominates M7

**The engine's embedding surface is already built as a Rust API on
`doodle-core`, and the wasm facade already exercises almost all of it** — so M7
is *mostly* marshalling. But the review sharpened where it is **not**, and those
seams carry the milestone's real risk:

- **Already built + exercised (true marshalling):** instance lifecycle,
  `drive`/`resolve`/`cancel`, the six-way `Outcome`, handles, output, positions,
  the E§8 observation surface, capability registration, native modules / foreign
  functions (`machine/native.rs`, the S-42-lite descriptor), non-local exits
  (S-46), destroy/GC finalization (`machine/foreign.rs`).
- **Genuinely new mechanisms (not marshalling — the risk):** (1) the
  **`#[repr(C)]` ABI-freeze discipline** the codebase has never needed (no
  `repr(C)` exists today); (2) the **control-inverting `doodle_call_block`**
  (the engine calls a C callback that calls back in — the one real `&mut`
  aliasing hazard); (3) **GC-time finalizers** in C form; (4) the **first
  end-to-end suspending capability** (`read_line`) and the **cross-surface
  conformance harness** that certifies trace identity.

AD1: **`doodle-capi` is the only crate permitted `unsafe`.** Every pointer
crossing is a soundness obligation concentrated there.

## M7 exit criteria (accept, from the M7 paragraph)

- A **~300-line C host** embeds the engine, registers primitives — **including a
  foreign function with a default parameter and a block parameter, both binding
  per L§8.3** — and runs the conformance suite through the C ABI with **traces
  identical to the wasm surface** (transitively: both match each fixture's
  canonical expected transcript — see the trace schema, M7.5).
- **Two instances on two threads** run independently with **independent
  deterministic traces**, and a **cross-instance handle use is caught in debug
  builds**.
- **Sanitizers clean**: ASAN/LSAN/UBSan on the C host; **Miri on the C ABI's
  Rust side** (see D-M7-11 — Miri must exercise the `unsafe`, which lives in
  `doodle-capi`, not `doodle-core`).
- **`doodle run`** executes gallery programs (incl. failure presentation); the
  suite is **green through all three surfaces** and stays green (a standing gate,
  not a one-time snapshot).
- npm/crates **publish dry-runs** pass (per D-M7-10 on what actually publishes).

## Decisions

### Resolved

**D-M7-1 — R4 / foreign-yield (E§5.4, App C S-15). RESOLVED (user, 2026-09-02;
discussions `c377b32`).** **Forbid-and-fault is the frozen v1 ABI rule** — a
suspending capability inside a foreign/native callback (a block consumer's
reentrant drive) is a terminal `NestedSuspend` fault. The freeze is
**additive-open**: the designated extension is a future descriptor flag
(`resumable`) opting a callback into a yield protocol, unflagged callbacks
keeping fault semantics forever (loosening only). The idiom for suspension inside
iteration is **inversion** — expose suspending *capabilities*, let Doodle own the
loop — now normative in E§5.4. **M7 obligations:** ship forbid-and-fault (no
yield protocol); the conformance suite carries the `NestedSuspend` fault fixture
**and** a Doodle-consumer inversion **parity control**; App C S-15 closes fully.

### Resolved — D-M7-2..11 RATIFIED (user, 2026-09-02)

All ten ratified as recommended, with **two adjustments** (D-M7-8: the
captured-constant default is *restricted to transitively immutable values*,
making the divergence vanish; D-M7-10: dry-run-ready with publication
deferred to D-8) and **two riders** (D-M7-2: the payload-only finalizer is
also the E§11 determinism guard; D-M7-9: conventions platform-neutral even
though certification is Linux-only). D-M7-7's builder shape is already
vindicated: the per-op result-size cap just added to Limits would have
broken a frozen by-value config struct.

**D-M7-2 — S-42 foreign-function descriptor + finalizer, C form (E§5.1).**
- **Descriptor shape → opaque + builder functions** (review-preferred over a
  by-value struct): `doodle_foreign_desc_new()` / `…_set_default(slot, handle)` /
  `…_set_block_param(slot)` / (future) `…_set_resumable()`. This is the cleanest
  **additive-open** shape (D-M7-1) and sidesteps by-value struct-size ABI breaks.
- **Defaults** are captured **value handles** materialized into the instance heap
  as a **`ConstValue` recipe rooted at load** (not a raw registry-held `Value` —
  the registry is built before any heap exists and is deliberately **not a GC
  root**; a heap-backed default there would dangle at the first GC). *Evaluation
  timing is D-M7-8.*
- **Block parameter** is delivered to the callback as the callback's **`ctx`
  handle**, invoked via `doodle_call_block(ctx, …)` — **never** the instance
  pointer (that would create a second live `&mut Instance` = UB). `doodle_call_block`
  returns a status covering **Completed / NonLocalExit / Raised / Faulted**;
  after a `NonLocalExit` (S-46) the callback **must return promptly** with no
  result, or the instance faults (host-contract, E§7.6).
- **Finalizer** is `extern "C" fn(void*)` (abort-on-unwind, **not**
  `extern "C-unwind"`), run at GC **and** at `destroy` (E§3.1/§4.5), receiving
  **only** the `void*` payload — never the instance — so re-entry is
  **structurally impossible**, not a debug-asserted rule. **Rider (ratified):**
  this is also the **determinism guard** — finalizers run at GC time, GC timing
  must never be Doodle-observable (plan §4.1/E§11), and a finalizer that cannot
  touch the instance cannot feed GC timing back into observable state.

**D-M7-3 — The frozen C surface: ABI discipline (the stable-surface freeze).**
- **Dedicated `#[repr(C)]` ABI types owned by `doodle-capi`** with explicit
  discriminants (`DoodleKind`, `DoodleOutcome`, `DoodleStatus`, `DoodleFault`,
  …), hand-mapped from `doodle-core`'s default-repr enums — exactly as
  `facade.rs` hand-maps to string tags. cbindgen parses **only `doodle-capi`**
  (no `parse_deps`), so the header can't leak internal representation.
- **Opaque instance** (`DoodleInstance*`); **handles `uint64_t`** (per-instance);
  **strings `(const uint8_t* ptr, size_t len)` UTF-8**, never NUL-terminated;
  **outcome a tagged struct** with a **reserved tail + an unknown-tag value** so
  a new arm (`SuspendedImport` is the day-one proof it works) fits without
  resizing; fallible calls return a **`DoodleStatus`** + out-params.
- **Symbol hygiene**: cbindgen `[export] prefix = "Doodle"`, `[enum]
  prefix_with_name`, explicit enum reprs, named fn-ptr typedefs
  (`DoodleFinalizer`/`DoodleResolver`/`DoodleForeignFn`). No bare `Int`/`String`/
  `Type` constants in the global namespace.
- **ABI version distinct from the engine version**: a `DOODLE_ABI_VERSION_MAJOR/
  MINOR` header macro + `uint32_t doodle_abi_version(void)` (major = break, minor
  = additive) — the version the `resumable` extension advertises under.
- **Compatibility = binary-compat** (adopt the stricter cdylib rules now, even
  shipping only a staticlib, so the Cargo.toml's anticipated cdylib doesn't turn
  every by-value size change into a runtime break).
- **The regen-no-diff gate proves currency, not compatibility** — add an
  **additive-only** check (a committed golden header compared for removals/
  signature-changes/discriminant-renumbering, additions allowed).
- **M7.1 freezes the *conventions*** (the repr rule, discriminant-assignment
  rule, reserved/version rule, `Doodle*` prefix rule, string-ownership rule) as a
  written checklist; M7.2/M7.3 mechanically apply them.

**D-M7-4 — CLI scope; `doodle test`; and S-19 (Critical).**
- `doodle run <file>` runs to completion (FS module resolution + CLI primitives),
  **rendering** an uncaught raise / engine fault / the S-63 load-diagnostics with
  **source spans** (not just the success path).
- **`time`/`random`/`read_line` are suspending capabilities (E§5.3), NOT
  synchronous foreign functions** — their values must cross the recordable
  boundary (E§11), or each surface computes them off-record and replay +
  cross-surface identity break. This is the **S-19** contract (sync FFs must be
  deterministic; anything reading a clock/RNG/input/external state MUST be a
  capability); documented at the `doodle.h` foreign-function descriptor.
- **`doodle test` in M7 = the conformance/drive-script harness** (wrapping
  `tools/conformance-runner`), **not** the M9a Doodle `test` module +
  `_test.doodle` discovery. Flagged so "the CLI has `test`" doesn't over-promise.

**D-M7-5 — Threading + cross-instance handle safety.**
- Instances are **independent, single-threaded each**; target **`Send`, not
  `Sync`**. `Send` is **not free today**: the core `Finalizer =
  Box<dyn FnOnce(u64)>` lacks `+ Send`, so `Instance` is `!Send`. **M7.0 widens
  it to `+ Send`** (the C finalizer captures only the `void*` payload + fn ptr)
  and adds a compile-time `assert_send::<Instance>()`. `!Sync` is correct
  (`StrObj.graphemes: OnceCell` is `&self`-mutated).
- **Cross-instance handle use caught in a debug build** via a **capi-side table**
  mapping each `DoodleInstance*` to the handles it minted (keeps the clean
  `uint64_t`; works in **release** too, returning a status error — a raw
  `uint64_t` from instance A resolved on B would otherwise alias B's slot and,
  for a `Foreign`, hand a wrong pointer back to C: not "memory-safe" once it
  re-crosses to C). The generation-wrap (ABA) budget is documented (full 32-bit
  generation retained; no bit-stealing).

**D-M7-6 — String/byte ownership model (new).** *Recommend: copy-out by
default.* Return bytes/strings by **copying into a caller-provided buffer**
(`doodle_string_bytes(inst, h, uint8_t* out, size_t cap, size_t* needed)`),
matching what the wasm boundary actually does (it deep-copies to JS — there is no
interior pointer to "mirror"). Where a zero-copy read is worth it, specify a
**precise per-accessor validity window** (handle-backed reads: valid until
`release(handle)`/`destroy`; `output`: valid until the next `drive`/`resolve`/
`output`) — never a blanket "until the next call." Ratify the default.

**D-M7-7 — Descriptor/config extensibility mechanism (new).** *Recommend:
opaque + builder functions* for the foreign-function descriptor **and** the
create/config surface (`doodle_config_new()`/`…_set_limits()`/`…_set_observation_mode()`/
`…_set_target_unicode_version()`/`…_set_module_resolver()`/`…_set_host_data()`).
The config surface is host-facing and by-value config structs can't grow without
an ABI break; builders make the whole extension story additive. The config
**must** carry, at freeze: limits, drive granularity, observation mode, the
**target Unicode version (S-41 — the replay guard; omitting it reopens the silent
grapheme/normalization divergence S-41 closed)**, the module resolver, and
**host-data** (E§5.2 — foreign functions receive it).

**D-M7-8 — S-42 default *evaluation timing* vs L§8.3 (new — a semantics
question).** L§8.3 governs how Doodle-callable defaults bind. A foreign default
captured as a constant value handle is **constant per call**; if L§8.3 evaluates
a default expression **per call**, the foreign form diverges (constant vs
per-call; shared-mutable-default aliasing). The four argument-error kinds (S-58)
are unaffected (defaults fill *missing* args; S-31/S-42 share the binding path).
**RATIFIED (user, 2026-09-02) with an adjustment that makes the divergence
vanish rather than accepting it:** a foreign default must be a **transitively
immutable value** (the S-29 class — value-typed, no reachable shared mutable
content); the descriptor builder **rejects** a List/Dict/ref-record default at
build time. For immutable values, evaluate-once and evaluate-per-call are
indistinguishable (L§4.13/§4.14 — identity is observable only for reference
types), so **L§8.3 is preserved, not diverged from**; without the restriction
the C surface would import Python's mutable-default-argument footgun into the
one place (host code) a kid cannot inspect. The M7.0 E§5.1 sentence states it
as "constant *because* constrained immutable, hence L§8.3-equivalent".

**D-M7-9 — C-ABI certification platform scope (new — flagged carve-out).**
*Recommend: Linux-only for M7.* The `capi`/C-smoke CI runs ubuntu-only and
ASAN/LSAN/UBSan are Linux/clang-centric; M7 would certify the C surface on
**Linux only**, with macOS/Windows C embedding + sanitizer coverage deferred.
**ACCEPTED (user, 2026-09-02) with a portability rider:** the M7.1 freeze
conventions must be **platform-neutral** — no gnu-only attributes, no
platform-varying sizes/alignments in `doodle.h` — so the carve-out defers
*testing*, not portability.

**D-M7-10 — Distribution / what publishes (new — flagged).** All three crates
are `publish = false`, so `cargo publish --dry-run` errors on each. `doodle-capi`
is a **staticlib** — not a crates.io Rust dependency; its real artifact is
**`doodle.h` + a built archive/shared lib**, not a crate. The npm
`@doodle-lang/*` package lives in the **`doodle-web` submodule** (cross-repo).
**DECIDED (user, 2026-09-02):** the C artifact is **header + built lib + the
embedder README**; npm stays in `doodle-web`; `doodle-core` + the `doodle` CLI
become **dry-run-ready** (publish keys set — the CLI transitively requires
core on crates.io — with an explicit "internal API, no stability promise; the
C ABI is the stable embedding surface" disclaimer in `doodle-core`'s README);
**actual publication waits for D-8** (release cadence, an open §10 decision
the user owns). M7.7's dry-run criterion is thereby scoped honestly without
pre-deciding distribution posture.

**D-M7-11 — Miri target + sanitizer split (new).** Miri must exercise the
`unsafe`, which AD1 concentrates in `doodle-capi` — but that crate is
`staticlib`-only and not `cargo test`-able. Add `crate-type =
["staticlib", "rlib"]` (or a sibling test crate) so the `extern "C"` functions
are callable from Rust `#[test]`s, and run **`cargo miri test`** against those
(forged/cross-instance/double-released handles, the re-entrant block path,
returned-pointer windows). Split: **Miri = Rust-side aliasing/UAF on the capi**;
**ASAN/LSAN/UBSan = C-host misuse + foreign-resource leaks**. (Miri on
`doodle-core` alone certifies the wrong crate — it has no `unsafe`.)

**Also settled (no decision):** suspend-the-outer-drive is out (D-M7-1); live
edit stays out (§1.2); the CLI is a **Rust binary over `doodle-core`** while the
**example C host** exercises the C ABI (different consumers by design — but see
M7.6's standing gate, so the C surface can't rot).

## The work items

### M7.0 — Engine pre-work: S-42 close, `Send`, finalizer, defaults `[M]` (D-M7-2, D-M7-5, D-M7-8) — DONE (doodle-rust `8ded450`)

Pure engine + spec, no C yet. Land the **S-42** E§5.1 edit (descriptor defaults +
block-param + the `extern "C" fn(void*)` finalizer). Widen `Finalizer` to
`Box<dyn FnOnce(u64) + Send>`; add `assert_send::<Instance>()`. Make heap-backed
defaults a **`ConstValue` recipe materialized + rooted at load** (not a
registry-held `Value`); GC-stress test collecting while a default-bound call is
in flight. Land the ratified D-M7-8 rule: defaults are captured constants **restricted
to transitively immutable values** (builder rejects mutable ones), stated in
E§5.1 as L§8.3-equivalent by construction. Tests: a foreign function with a default + block parameter
binding per L§8.3; a foreign value finalized exactly once at **both** GC and
`destroy`.

### M7.1 — C ABI core + the freeze conventions `[M–L]` (D-M7-3, D-M7-6, D-M7-7) — DONE (doodle-rust `07b265e`)

> Landed: the `#[repr(C)]` ABI-type mirror + cbindgen prefixing + `doodle_abi_version()`;
> opaque config builders (limits / observation mode / S-41 target Unicode version);
> `doodle_load`/`doodle_free`; `doodle_drive`/`doodle_drive_slice`/`doodle_cancel` + the
> flat six-kind `DoodleOutcome` (reserved tail); `doodle_raised_kind`/`_message`/
> `doodle_output` (copy-out); the value boundary routing through the canonicalizing/NFC
> constructors; a `catch_unwind` firewall at every boundary; `crate-type += rlib` +
> `tests/abi.rs`; the C smoke host does a load/drive/handle round-trip. S-19 landed (E§5.2).
> **Deferred to M7.2 (its natural home, flagged not silent):** the additive-only golden-header
> gate (the regen-no-diff gate is in place; the removal/renumber check rides with the first
> real extension), and populating `DoodleOutcome::value` for a reentrant `fn` return (needs an
> intern-to-handle path, M7.3).

The highest-stakes chunk. Establish the **`#[repr(C)]` ABI-type mirror**,
cbindgen prefixing/config, the ABI-version macro + `doodle_abi_version()`, the
additive-only header gate, and the **written freeze-conventions checklist** (platform-neutral per the
D-M7-9 rider: no gnu-only attributes, no platform-varying layout).
Then the core surface over `DoodleInstance*`: create/free; **config via builders**
(limits, granularity, observation mode, **target Unicode version**, resolver,
host-data); `drive`/`resolve`/`cancel`; the tagged **outcome** (with
`SuspendedImport` present day one, reserved tail, unknown-tag); handles
(`make_*` **routing through the canonicalizing `Instance::make_float`/
`make_string`** — canonical NaN, NFC — with an inject-signaling-NaN/non-NFC
boundary test) + `as_*`/`string_bytes`/`kind_of`/`release` per the **copy-out**
model (D-M7-6); `output`; positions; `describe_raised`. **`catch_unwind` at every
`extern "C"` boundary** → `DoodleStatus` (panic=unwind default makes an escaping
panic UB). Regenerate + commit `doodle.h`; extend the C smoke test. Freeze the
conventions here, before M7.2/M7.3 replicate them.

### M7.2 — C host extensions: capabilities, foreign functions, resolver `[M–L]` (D-M7-2, D-M7-1) — DONE

Landed in sub-pieces (each gated + CI-green):
- **M7.2a** (doodle-rust `6eb77cc`) — capability registration + resolve + the
  engine built-ins by identity (`DoodleRegistry`, `doodle_registry_add_builtin`).
- **M7.2c/d** (doodle-rust `9c03111`) — the **module-resolver** callback (import
  path → source / not-found / raise) and **foreign values** + the
  `extern "C" fn(void*)` finalizer, both pull-based (no engine change).
- **M7.2b** — the control-inversion piece. **Step 1** (engine, `a2d40a4`):
  `ForeignBody::Host` + the curated `pub` `IntrinsicCtx` host-call API
  (`arg_handle`/`invoke_block_handles`/`emit`/`fault_host`) + `ForeignBuilder`.
  **Step 2** (C ABI + the full in-callback value API, `525c294`): the opaque
  `DoodleForeignDesc` builder, `doodle_registry_add_foreign`, `DoodleCallCtx` +
  `doodle_call_arg`/`block`/`emit`/`set_result`/`set_raise` (result/raise handles
  **consumed**), and the ctx-based `doodle_call_make_*`/`as_*`/`kind_of`/`list_*`/
  `foreign_*`/`release` (so a `fn` callback reads args + builds a result). The
  value ops were refactored to a shared core (`machine/values.rs`) that both
  `Instance` and `IntrinsicCtx` delegate to — this also root-caused a determinism
  gap (`materialize_const` now canonicalizes a `ConstValue::Float` NaN, S-28).
- `doodle_call_block` routes through the callback's `ctx` (never a second
  `&mut Instance`), returns Completed/NonLocalExit/Halted, and enforces the
  **S-46 return-promptly** + **S-15 `NestedSuspend`** backstops.

**Decisions taken (reported, ratified-by-default):** the callback speaks in
*handles* (`HostReply`), so the C ABI never names an engine `Value`/`Raise`
(`Arc<dyn Fn>`, not `Box`, keeps `ForeignBody: Clone`); `set_result`/`set_raise`
**consume** their handle (the host can't reach the ctx after return); the full
in-callback value API was built (user chose it over an accept-only surface).

**FFI-soundness review (adversarial, read-only) caught a CRITICAL** before land:
the original `live: bool` gate rejected a returned ctx but not an *ancestor* ctx
whose block is mid nested-drive → a reentrant host touching a stashed ancestor
ctx formed a second aliasing `&mut IntrinsicCtx` (UB). Fixed with a thread-local
**innermost-ctx** gate (pointer-equality, never derefs before confirming
innermost); the re-review confirmed both CRITICAL and MAJOR (freed-stack `live`
read) closed, **verified under Miri/Stacked Borrows** (all 22 ABI tests pass,
incl. the reentrant-ancestor regression test). Tests: foreign fn with
default+block; a `fn` reads args + constructs a result; a callback raises a
ctx-built value; the reentrant-ancestor touch → `ErrContract`; native
re-drive/return-value-after-`NonLocalExit` → `Faulted`. The C smoke drives a
foreign `greet()` from real C. (Full Miri sweep is M7.6/D-M7-11.)

### M7.3 — C observation/debug surface `[M]` — DONE

Mirrors the E§8 pull surface into the C ABI. Four ABI-shape decisions ratified
(D-M7-12..15) and captured in `plan/m7.3-observation-design.md`; landed in chunks
(each gated + CI-green):
- **M7.3a** (doodle-rust `eae226d`) — pause-generation core + positions
  (`doodle_current_position`/`_completed_position`/`_current_result`), stack walk
  (`doodle_stack_frame_count` → count + generation; `doodle_frame_at`, gen-checked
  before bounds → `ErrStale`/`ErrIndexOutOfBounds`; lazily-minted `doodle_frame_callable`),
  and `doodle_module_canonical_id` (opaque module token → canonical id). **Also
  found + fixed a MAJOR null-handle collision** (a live handle could encode to `0`
  == `DOODLE_NULL_HANDLE`; generations are now 1-based) and closed the deferred
  `DoodleOutcome::value`-on-Completed item.
- **M7.3b** (`d2715f5`) — frame local/dynamic bindings + module globals (count +
  per-slot name + lazy value handle), all gen-checked.
- **M7.3c** (`a666d97`) — structural value inspection (record/dict/list/callable/
  type/module-member) + `doodle_eval_to_string`; read by handle, not pause-scoped.
- **M7.3d** (`b979f79`) — breakpoints, raise-trap, host pause, runtime observation
  mode, tail-elided history (gen-checked), and the S-63 load-diagnostics pull
  record.
- **M7.3e** — the C smoke drives a paused-stack walk from real C; the wasm facade
  already reports staleness with a distinct `StaleGeneration` error, matching C's
  `ErrStale` (the D-M7-12 parity rider — no wasm change needed).

**All value rendering routes through the engine's structural inspection** (never
handle ids / host formatting / raw foreign `ptr` / module tokens in host-visible
output). Handle discipline (host-owned + released, no live handle encodes to `0`)
was the review focus (a read-only handle-discipline pass). Three-surface parity of
the observation *results* is exercised in M7.5.

**Deferred to the M7 spec-reconciliation** (captured in the design doc): the E§8
staleness contract + opaque-module-token text; App C S-24 (module token ⇄
canonical on M9b reload) + the `ErrStale`/`StaleGeneration` cross-surface
vocabulary; module tokens on the M7.5d trace-schema exclusion list; the embedder
README's host↔engine addressing-asymmetry note.

### M7.4 — The `doodle` CLI `[M]` (D-M7-4)

A new `doodle` binary crate over `doodle-core` (not through the C ABI; placement
per AD7): `doodle run <file>` (load → drive, streaming `print`, **rendering
raises/faults/S-63 diagnostics with spans**); **FS module resolution** (resolver
→ sibling `.doodle`, singleton by canonical path, L§11.3); the CLI **primitive
capabilities** `print`/`read_line`/`time`/`random` — **`read_line`/`time`/`random`
as suspending capabilities** whose resolutions are replay inputs (D-M7-4/S-19);
`doodle test` wrapping the conformance runner. Name the **minimal gallery** M7.4
runs (the `turtle.doodle`-style programs; the full examples gallery is M10).
Tests: CLI integration over the gallery; a multi-file program resolves; a scripted
`read_line`.

### M7.5 — Conformance substrate + example host + three-surface parity `[L]` (Critical — was a parenthetical in the draft)

The apparatus that delivers the headline accept criteria. **(a)** Extend the
drive-script grammar + `drive.rs` with **capability-resolution steps** (the slots
M6.8 deferred here — `read_line` is the first real suspending capability) and a
**foreign-function/native-module registration declaration**. **(b)** Define the
**portable conformance test-capability registry** installed **identically** by
native, wasm, and C (registration order is replay identity, E§5.5 — the C
example host and the wasm conformance host share one **ordered manifest**).
**(c)** Author the fixtures the accept criteria name: the S-42 default+block
foreign fn, the **`NestedSuspend` fault + Doodle-consumer inversion parity
control** (D-M7-1), and `read_line`/`time`/`random` capability fixtures.
**(d)** Pin the **cross-surface trace schema** — positions, outcome, stack frames
with **structurally-rendered values**, output bytes, capability request identity
+ resolution — explicitly **excluding** handle ids, foreign `ptr`s, host-side
float formatting, and any HashMap-order dependence; C↔wasm identity is
**transitive** via each fixture's canonical transcript (no direct C-vs-wasm diff).
**(e)** The **~300-line example C host** + the C-surface conformance mechanism
(Rust orchestrator does discovery/parse/compare; a thin C driver drives one
fixture and emits a transcript). **(f)** Make conformance-through-C a **standing
CI gate** (surface it for the user to hook, like the wasm gate — do not self-wire).

### M7.6 — Threads, cross-instance guard, sanitizers, Miri `[M–L]` (D-M7-5, D-M7-9, D-M7-11)

Two instances on two OS threads with independent deterministic traces; the
**cross-instance handle guard** (capi side-table, D-M7-5) with its debug/release
test. **ASAN/LSAN/UBSan** on the C host; **`cargo miri test`** on the capi rlib
(D-M7-11) — budget the **first Miri bring-up** on a now-large core (AD8's M2a
Miri never landed). Extend the GC-stress determinism gate to the C surface.
Platform scope per D-M7-9. Wire each as its own CI job (surface; don't self-wire
beyond what's asked).

### M7.7 — Publish dry-runs, distribution, exit review + close `[M]` (D-M7-10)

Per D-M7-10: the publish dry-runs for whatever actually publishes; the **C-ABI
distribution artifact** (`doodle.h` + built lib) and an **embedder README**
(build/link recipe, handle-ownership + string validity-window contract); note the
npm dry-run's cross-repo (`doodle-web`) location. Multi-lens **adversarial
review** (read-only) with an explicit `unsafe`/FFI-soundness + ABI-stability
lens. App C discharge: **S-42** resolved, **S-15** fully closed, **S-16/S-46**
re-verified in C form, **S-19** discharged, **S-41** carried into the C config.
Update the M7 paragraph status + `claude-todo`.

## Notes on ordering and risk

- **M7.0 → M7.1 → {M7.2, M7.3} → M7.4 → M7.5 → M7.6 → M7.7.** M7.0 (Send +
  finalizer + defaults + S-42 spec) and M7.1 (the frozen conventions) gate
  everything; M7.2/M7.3 apply the conventions; M7.5 depends on the whole surface
  + the CLI's first capability and is the crescendo; M7.6 is orthogonal once the
  surface exists.
- **The freeze conventions (M7.1) are the highest-stakes decision** — `doodle.h`
  is a binary-compatibility promise, and the one extension D-M7-1 mandates (the
  `resumable` flag) is only additively-addable if the descriptor/config/outcome
  shapes carry a growth mechanism (opaque builders + reserved/versioned tags).
  Get them right before M7.2/M7.3 replicate them across dozens of symbols.
- **`unsafe` is concentrated by design (AD1) but genuinely hard in two spots:**
  the control-inverting `doodle_call_block` (`&mut` aliasing) and returned-pointer
  validity. Miri-on-the-capi-rlib (D-M7-11) is the gate that actually certifies
  them; ASAN/UBSan do not understand Rust aliasing.
- **Determinism is the acceptance bar, and the C path opens new leak vectors:**
  off-record sync FFs (S-19), host-side value formatting, HashMap order, and
  handle-id-in-trace. The trace schema (M7.5d) + the constructor-canonicalization
  (M7.1) + registration-order parity (M7.5b) close them; any divergence is a
  release blocker (E§11).

## Spec-delta obligations coming due in M7

- **S-42** (E§5.1) — descriptor defaults + block-param + `extern "C" fn(void*)`
  finalizer. **Resolve in M7.0.**
- **S-19** (E§5.2/§11) — synchronous-foreign-function determinism host-contract
  ("pure/deterministic, or become a capability"). **Land the E§5/§11 sentence in
  M7.0/M7.1**; it is the contract the frozen sync-FF surface first exposes to
  un-reviewable hosts.
- **S-41** (E§3.1) — the target Unicode version is a **frozen config field**
  (the replay guard); ensure it's in the M7.1/M7.7 surface, with a create-time
  mismatch test.
- **S-15** (E§5.4, R4) — **M7 half already closed** (`c377b32`); M7.7 marks full
  closure.
- **S-16 / S-46** (E§5.4/§7.6) — abandoned nested drives + non-local-exit
  host-contract, in their **C-boundary form** (`doodle_call_block` return codes +
  return-promptly fault); verify in M7.2, re-confirm M7.7.
