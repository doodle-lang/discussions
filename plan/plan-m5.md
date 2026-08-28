# Plan — M5: Modules, imports, protocols, prelude `[L]`

Working plan for **Milestone M5**, session-sized. Written when M5 begins (M4
complete, 2026-08-26). Scope source: `implementation.md` M5 paragraph, **AD5**
(name resolution: lexical slots + module-scope binding cells), §3.4/§3.6/§3.8,
and App C **S-8/S-13/S-14/S-31/S-32/S-39/S-44**. Specs: **L§10** (protocols),
**L§11** (modules/imports), **L§4.11–§4.12**, **L§15** (well-known protocols),
**E§6** (module loading), **E§5.5/§13** (native modules, platform primitives).

## The one fact that dominates M5

The engine is still **single-module**: `Instance` holds one
`resolved: ResolvedModule` and one flat `namespace`, threaded as singletons
through `step`/`control`/`gc`. The **front end already parses** the full M5
grammar — `Protocol`, `Implement`, `Exports`, `Import`/`ImportTarget`, `Module`
AST nodes exist — so M5 is almost entirely a **resolve + machine + drive-layer**
milestone. Its real cost and risk is the **single-module → multi-module
refactor** (M5.0), not the language surface. Everything else sits on top of it.
M5 carries the plan's widest error bars (Appendix A).

## M5 exit criteria (accept, from the M5 paragraph)

1. A **three-module project** where user code's `with pen_color = …` changes
   drawing done *inside* the imported turtle wrapper (the **cell-aliasing**
   proof).
2. **Assignment to an imported name** is rejected statically with a
   provenance-naming diagnostic (S-39).
3. A deliberate **wildcard collision** produces the two-module ambiguity error
   **on use** (S-13).
4. A module **raising at load** is `failed`, and re-import re-raises (S-8).
5. **`import` of a module with a missing platform primitive** fails per E§13.
6. The **M4 carve-out clauses go green**: protocol-`is`, and interpolation over
   **real** `Stringable` dispatch (not the placeholder renderer).

## Decisions needed (the user's calls — resolve before the dependent chunk)

Recommendation first; each blocks only the chunk(s) noted. **None block M5.0**
(a pure internal refactor). Already-resolved deltas M5 rides on are listed after.

1. **★ D-M5-1 · Stringable/Hashable — the M5 vs M9a split.** *Blocked M5.7.*
   **RESOLVED (user, 2026-08-27; implemented in M5.7).** The engine natively
   **defines** `Stringable`/`Hashable` at instance load — each a single member
   (`to_string`/`hash`) with a **native default** (the renderer / the structural
   hasher) — and binds the names `Stringable`, `Hashable`, and the `to_string`
   dispatcher into every module's prelude (folded into the M5.8 prelude import;
   `hash` is *not* a bare name — the engine hash isn't build-stable). The two
   decisions taken:
   - **(Q1) Compound/records → M9a.** Scalars render/hash finally; a `list`/`dict`/
     `record` with no explicit `implement` keeps the provisional `<list>`/
     `<record>` render placeholder and the native structural record hash. The
     Doodle-written record-default `to_string`/`hash` and real compound rendering
     land at **M9a**.
   - **(Q2) Both are real user-implementable dispatch NOW.** Interpolation drives
     an explicit `implement Stringable`'s `to_string` (a real, can-raise call,
     resumed by a `StrInterpRendered` continuation); dict insert/lookup drive an
     explicit `implement Hashable`'s `hash` (resumed by the `*Hashed`
     continuations). Absent an explicit impl, both fall to the native default.
   - **Provisional riders (documented, revisited at M9a):**
     - **Interpolation-only driven render.** `print(x)` and error/host-raised-value
       rendering keep the native seam (never drive user code mid-error; `print` is
       a demo intrinsic). So `print(point)` shows `<record>` while `"{point}"` shows
       the user's text.
     - **`is` reflects native coverage.** `x is Stringable` is **true for every
       value** (the native renderer is total); `x is Hashable` is true iff the
       value is natively hashable (`check_hashable`) **or** has an explicit impl —
       so `[1,2] is Hashable` is `false`. Built-in types are not *registered*
       `Stringable`/`Hashable` implementors (their L§15 stdlib impls land at M9a),
       but the native default makes `to_string(5)` and `"{5}"` agree.
     - **Native-protocol `implement` conformance is unchecked.** `implement
       Stringable/Hashable for T` is invisible to the same-module resolver (the
       protocol isn't in the AST), so a malformed one (wrong arity, stray method)
       is silently registered — the same gap as any cross-module `implement`,
       whose S-31 structural check routes to a load-time `module-load-error` and
       is not yet built. Track with the S-31 cross-module conformance discharge.
2. **★ D-M5-2 · Can the host module resolver suspend?** *Blocks M5.1.* E§6 stated
   `resolve(module_path) → Source | NotFound` **synchronously**; a browser host
   fetching over the network needs async. **RESOLVED (user, 2026-08-27; spec
   LANDED — App C S-60): async-capable from v0.1, as a SUSPENSION.** `import`
   suspends with `ImportRequest(path, importer)`, resolved with `Source` /
   `NotFound` (→ `module-not-found`) / `Raise(h)`; a bundling host resolves
   immediately in its drive loop (the trivial case, not a mode). Reuses M2b.4's
   parking/resume; import resolutions enter the replay record; S-15 never
   applies (imports are top-level statements). Synchronous-for-v0.1 rejected:
   it saved almost nothing and would have frozen the wrong shape into the M7
   C ABI. M5.1 implements.
3. **★ D-M5-3 · S-44 · May native modules declare `parameter` cells?** *Blocks
   M5.4/M5.9.* The plan **assumes not** — dynamic parameters live in **Doodle
   wrapper modules** over native primitives (the turtle wrapper declares
   `parameter pen_color`). **RESOLVED (user, 2026-08-27; spec LANDED — App
   C S-44): confirmed and generalized** — a native module exports foreign
   functions/consts/foreign values/records and nothing else (no `parameter`
   cells, protocols, or implementations); dynamic state reaches a native
   function as an argument from its Doodle wrapper. M5.4 enforces the member
   kinds at registration; M5.9's turtle wrapper is built on it.
4. **★ D-M5-4 · `extends` semantics depth (L§10.1 under-specified).** *Blocks
   M5.5.* L§10.1 pins `extends` as a *requirement* relationship with parent
   defaults reaching child implementors "through the ordinary default-member
   mechanism," but does not pin multi-level default-resolution order, diamond
   legality, or `extends`-cycle handling. **RESOLVED (user, 2026-08-27; spec
   LANDED — App C S-61): linear chain, nearest wins.** Transitive requirements;
   re-declaration (strengthen/override) with a conforming signature; an explicit
   impl beats any default, else the nearest declaring protocol's default; a
   would-be cycle is unwritable (forward `extends` → `used-before-defined` at
   load; no detection); diamonds impossible (single parent) — a shared ancestor
   counts once and chains never trigger §10.3's ambiguity; one `implement`
   block covers the chain. M5.5 implements.
5. **★ D-M5-5 · `module … end` block form vs file-level module (S-14 / L§11.1
   provisional).** *Blocks M5.3.* L§11.1 marks the file-level/block-level
   interaction provisional. **RESOLVED (user, 2026-08-27; landed M5.3b; L§11.1 +
   App D.1 updated by the spec author):** a sole file-wrapping `module Name …
   end` is unwrapped (`Name` documentation-only; identity = path); any other
   `module` block is the static `nested-module`; in-file sub-namespace modules
   deferred past v0.1. S-14 closed.
6. **★ D-M5-6 · S-43 parked shadowing warning (non-gating).** *M5.8 polish.*
   Should declaring a name that hides a prelude name (`let print = 5` then
   `print("hi")` → NotCallable) fire the L§5.1 shadowing warning? Applies to
   type values and intrinsics alike; needs the front end to know the prelude
   name set (natural once the prelude is an import). **RESOLVED (user, 2026-08-28; L§5.1
   landed): yes, warn** — wildcard-supplied names, prelude included, are outer
   bindings for the shadowing warning. Implementation **slips deliberately** to
   a post-M5.8 follow-up, **due before M6**: a post-resolve load-time diff of
   `ResolvedModule.globals` against the prelude's exports (registered before
   first `load`) — no resolver-API change. User-wildcard shadowing: import-time
   or linter, later.

**Already RESOLVED — M5 rides on these (cite, don't re-decide):**
**S-39** (imported names are live **read-only** aliases; assignment is a static
error; `with`-binding an imported parameter is legal — M5 residue is only
wildcard provenance-naming). **S-13** (wildcard-collision: second wildcard marks
ambiguous; use raises naming both; explicit/selective override).
**S-32** (native-module registration before first drive; mid-run deferred).
**S-8** (load failure = `failed` + re-raise; reload is environment-level, M9b).
**S-52** (protocol member `end`; empty=required, non-empty=default — *implemented*).
**S-45/S-55** (block-arg call is not a tail; tail reuse needs kind match —
*already discharged*, no M5 action).
**Open corners due *by* M5:** **S-14** (exports/module-block → D-M5-5),
**S-31** (protocol signature conformance + keyword-first-arg dispatch → M5.5).

## The work items

Ordering is by dependency. Sizes per Appendix A (S ≤ ~1wk, M ~1–3wk, L ~3–6wk).

### M5.0 — Multi-module machine foundation `[L]` (critical) — **CORE DONE (`6815e45`)**
- **Goal.** Turn the single-module machine into a multi-module one, with the
  current module as `ModuleId(0)`. No new language surface.
- **Landed (core).** `Instance` holds `modules: Vec<LoadedModule>` (resolved +
  namespace per module); **per-frame `Frame.module: ModuleId`** (callable →
  `CalObj`'s module, block → defining frame's module, top level → loading
  module, tail call updates it); `Instance::step` **derives** the executing
  frame's module's resolved + namespace (disjoint field borrows), so every free
  helper signature is unchanged; observe/lifecycle route through
  `current_resolved()`. All existing gates green; the single module is
  `ModuleId(0)`.
- **Deliberately deferred (become live only with a 2nd module / their
  consumers), each folded into the chunk that exercises it:** the **cross-module
  call lookup** (a callee's info lives in *its* module's resolved) and
  **multi-namespace GC rooting** (root all loaded modules during any module's
  step) → **M5.1**; **`CellObj` kind + provenance** → **M5.2** (provenance) /
  **M5.5** (dispatcher); the **module load-state machine** → M5.1.
- **Depends.** M4.
- **Tests.** All existing conformance (180) + determinism gates + wasm (104) stay
  green with the single module reframed as `ModuleId(0)`. (A two-module
  independent-rooting test lands with M5.1, when a second module can exist.)

### M5.1 — Resolver hook + load state machine + suspendable driving `[L]` (critical) — **M5.1a LANDED**
- **Goal.** Load a second module on first reference, drivably.
- **Landed (M5.1a — loading machinery).** The `import` **suspension** +
  `resolve_import(Source | NotFound | Raise)` host API (E§6/S-60); the
  `{loading, loaded, failed}` state machine + `by_path`/`by_canonical` singleton
  caches, with **cycle detection that survives a suspend**; driving an imported
  module's top level as **ordinary engine frames** (it can itself suspend);
  **circular-import diagnostic naming the cycle** (`a imports b imports a`); **S-8
  failed** — `LoadState::Failed(value)` retains the load's exception, re-import
  re-raises it unchanged (latent in a single run: a load failure is uncatchable
  and terminates — imports are top-level, `import`-in-`try` is a static error);
  `module-load-error` for a fetched module with static errors; the **multi-module
  GC rooting** the second module makes live (all modules' namespace cells are
  permanent roots). Two new S-58 slugs ratified: `circular-import`,
  `module-load-error`.
- **Still M5.1b / folded into M5.2.** The heap **module value** (name + source
  location + exported names/values + docstring, L§4.11/§13) — built once binding
  gives it a consumer; and the **cross-module call lookup** (a callee's info lives
  in *its* module's resolved) — live only once a bound cross-module call exists
  (M5.2). Name **binding** of every import form is M5.2.
- **Spec-delta.** D-M5-2 — **RESOLVED 2026-08-27, spec landed** (E§6 rewrite:
  import is a suspension; App C S-60; `module-not-found` slug). Implement the
  `Import` request kind per E§6/§7.5.
- **Depends.** M5.0.
- **Tests (M5.1a landed).** load-once singleton (top-level effect runs once);
  canonical-id dedupe; a 2-module cycle → the naming diagnostic; a module raising
  at load propagates + is marked `failed` retaining its exception (re-raise itself
  is latent — see S-8); a fetched module with static errors → `module-load-error`;
  a module whose top-level suspends (a capability at load) resumes with the importer
  parked; **the import itself suspends** — immediate (bundling) resolve, deferred
  resolve (importer parked, resumes), `NotFound` → `module-not-found`, `Raise(h)`;
  and the multi-module GC-rooting test. (`crates/doodle-core/tests/modules.rs` +
  machine unit tests.)

### M5.2 — Import forms + cell aliasing + provenance/ambiguity `[M–L]` (critical) — **DONE (a+b+c)**
- **Goal.** All import forms, with correct aliasing and read-only provenance.
- **Landed (M5.2a — the multi-module machine + bare-module form).** The
  **module-table threading** (`step`/`dispatch`/the call path + the reentrant
  nested drive take `&mut [LoadedModule]`; the executing module is derived from
  `resolved.canonical_id`); the **cross-module call fix** — `apply` reads the
  callee's parameters, defaults, slot layout, and body from **its own** module's
  AST (`CalObj.module`), the caller's from the call site; **`import m` / `import m
  as y`** binding (the module value, a new namespace cell added to the permanent
  GC roots); **`m.x` member access** (L§4.11 — a member is one of `m`'s own
  module-level definitions; prelude names in `m`'s namespace are not members;
  reads the live cell). Tests: cross-module call + defaults + live member read;
  `import`/`as`; missing-member and prelude-non-member raises; the bound cell
  survives forced GC. All prior gates green (conformance 180, wasm 104).
- **Landed (M5.2b — S-7 + member imports + cell aliasing).** **S-7** dotted-path
  resolution — the loader tries the **whole path as a module** first, and a path
  the host resolves `NotFound` (a `not_modules` cache) **falls back to member**
  (`import a.b` → member `b` of module `a`); a single-segment miss still raises
  `module-not-found`. **Member imports** (`import m.x` / `import m.x as z`) bind
  by **aliasing the exporter's cell** (AD5) — the importer's name maps to `m`'s
  existing binding cell for the member, so reads see `m`'s live value (proven by a
  cross-module-mutation liveness test); no new cell (the exporter's is already a
  GC root). A member that is not one of `m`'s own definitions raises `no-such-field`
  at the import. Tests: member const/fn/as, live alias, dotted-path-is-a-module,
  missing-member, missing-prefix. (Assignment to an imported name is already a
  static error, so S-39's core holds; its **provenance-naming** residue rides
  M5.2c.) Split `drive.rs` → `drive/config.rs` to stay under the length limit.
- **Landed (M5.2c — wildcard + S-13).** **`import m.*`** records a wildcard source;
  a free name not in the namespace **resolves on use** across the wildcard sources'
  exports (AD5) — 0 = undefined, 1 = a live alias, **2+ = `ambiguous-import`** raised
  at the use site naming both modules (import order). Explicit/selective imports and
  local decls are in the namespace, so they win for free (S-13 override). Slug
  `ambiguous-import` ratified (2026-08-27). Tests: wildcard-all-exports, live alias,
  explicit-override, local-shadow, **two-wildcard ambiguity (exit #3)**, undefined.
  (No `CellObj` kind tag needed for this — resolve-on-use handles provenance; the kind
  tag rides M5.9's `with`-binding of imported parameters / M5.5 dispatchers.)
- **Residue.** S-39's **wildcard-provenance-naming** for an *assignment* to a
  wildcard-imported name (the selective case already names its source) — polish, not
  gating; can slip to M5.8/M5.10.
- **Depends.** M5.0, M5.1.
- **Tests.** exit criteria #2 and #3 (M5.2c); each import form binds correctly; a
  `with`-bind of an imported parameter cell reaches the exporter's reads (M5.2b).

### M5.3 — `exports` + `module … end` corners `[S–M]` (parallel behind M5.1) — **DONE (a+b)**
- **5.3a (LANDED).** `exports a, b` restricts the public surface: the resolver builds
  `ResolvedModule.exports` (`None` = no `exports` statement = all public; union across
  multiple statements), and the three cross-module membership sites — `m.member` field
  access, `import m.member`, `import m.*` — consult it via `member_visibility` →
  {Exported, Private, Absent}. A private access raises **`not-exported`** `{module,
  member}` (loud-and-true, points at the fix); an absent one raises **`no-such-member`**
  `{module, member}` (the module container's access-miss kind — modules no longer reuse
  `no-such-field`); a wildcard's private name surfaces `not-exported` on the error path
  (not bare `name-not-defined`). An `exports` naming an undeclared name is the static
  **`undeclared-export`** `{module, name}` (adjective-first; the stale rubric row
  `assign-to-undeclared` → `undeclared-assignment` fixed while there). Files split:
  `machine/assign.rs` (assignment scheduling out of `control.rs`). Tests:
  `tests/exports.rs` (7). All gates green.
- **5.3b (LANDED).** The `module … end` block form — **★ D-M5-5 RESOLVED 2026-08-27:
  file-level `module Name … end` wrapping the file renames it; nesting deferred.** The
  resolver unwraps a sole file-wrapping `module` block (its body becomes the file module's
  top level, its docstring the module's; `Name` is documentation — no runtime effect); the
  machine runs the wrapper's body in the module frame. Any `module` block that is **not** the
  sole top-level statement (alongside other statements, or nested inside a wrapper) is the
  static **`nested-module`** (provisional — retired when sub-namespace modules land). Tests
  in `tests/exports.rs`: wrapper runs the body; non-wrapper and doubly-nested are static
  errors. **Spec:** L§11.1's "provisional interaction" note (and App D) should be updated to
  this resolution by the spec author.
- **Depends.** M5.1.

### M5.4 — Native modules + missing-primitive failure `[M]` (parallel w/ M5.5) — **DONE (a+b)**
- **Landed (a+b).** Host registration API: `Registry::register_module(NativeModule)` with a
  builder (`.function` / `.constant` / `.foreign` / `.record`) — the **full member set**
  (foreign functions, constants, foreign values, records; no `parameter`/protocol/implement,
  per S-44). Native modules are **pre-loaded** into the module table at instance creation
  (ids `1..=k`, replay-order) as synthetic modules (member-name `globals` + namespace cells
  of materialized values); `import`ing one finds it via the `by_path` cache and binds it
  through the existing M5.2 machinery — `m.member`, wildcard, cross-module call, record
  construction + `is`. Function members join the flat intrinsics in one `CallableTarget::
  Intrinsic` id space (appended after the prelude, seeding no module's prelude). A native
  module the host didn't register isn't in `by_path` → the import suspends → host `NotFound`
  → `module-not-found` (E§13 "primitives absent → fail to load"). S-32 is structural (the
  registry is consumed at load; `DuplicateModule` host error on a name clash). Tests:
  `tests/native_modules.rs` (8). Files split for length: `machine/native.rs`,
  `machine/intrinsic/ctx.rs` (extracted `IntrinsicCtx` + `apply`). All gates green.
- **Goal.** Host-provided native modules, consulted before source.
- **Lands.** Native-module registration (a member is a foreign function / const
  / foreign value / record), consulted **before** source lookup (E§5.5, §6);
  **S-32** timing (before first drive); an `import` of a module whose platform
  primitive is absent **fails per E§13**; the S-43 intrinsic path kept working
  (or folded into a native prelude module).
- **Spec-delta.** confirm S-32; D-M5-3 (S-44 — **RESOLVED 2026-08-27, spec
  landed**: no parameter cells/protocols/implementations in native modules;
  enforce member kinds at registration).
- **Depends.** M5.0, M5.1.
- **Tests.** exit criterion #5; a native-module member resolves and calls;
  registration after load is a host-API error.

### M5.5 — Protocols: dispatch, defaults, extends, qualified, ambiguity `[L]` (parallel track; critical for accept) — **DONE (a+b+c)**
- **Split.** **5.5a (LANDED):** `CellKind` tag (AD5, deferred since M2a) + protocol
  registry + `protocol`/`implement` load-time registration + dispatcher cells +
  single dispatch (S-31 bind-then-dispatch, positional **and** keyword) + defaults +
  qualified `P.member` + `protocol-not-implemented` + `ambiguous-member`. **`x is P`
  landed here too** (M5.6 absorbed — trivial once the registry existed).
  **5.5b (LANDED):** the **`extends` chain** at runtime (S-61) — parent resolved at load
  (parent-first), transitive requirements, chain-walk candidacy with ancestor subsumption,
  **nearest-default-wins**, `is` transitivity (`x is Child ⇒ x is Parent`); the
  dispatcher-value `is Procedure/Function` refinement (registry threaded into
  `types::callable_kind_of`). **Extends parent requirements enforced at runtime** (an
  unimplemented inherited required member raises `protocol-not-implemented` at the call).
  **5.5c (LANDED):** the **static conformance checks** — a resolver post-pass
  (`resolve/walk/protocols.rs`) over same-module protocols/implements, ratified slugs
  (user 2026-08-27): `dispatch-parameter-default` (first param no default),
  `protocol-signature-mismatch` (impl arity/block ≠ member; also a non-conforming
  re-declaration), `implementation-parameter-default` (impl restates a default),
  `incomplete-implementation` (missing required members of the chain, named + requiring
  protocol), `not-a-protocol-member` (a method the protocol doesn't declare). A
  cross-module `extends` parent / imported protocol is invisible to the resolver → its
  completeness checks fall to load (`module-load-error`, per spec).
- **Finding (S-61 cycle rule is vacuous — RESOLVED, spec reworded `8a11c06`):** with
  parent-first load ordering an `extends` target must already be a defined protocol; a
  forward/self reference reads an uninitialized cell → `used-before-defined` /
  `name-not-defined`. So an `extends` **cycle is unconstructable** (the neat proof, like
  S-46's import-in-try). The user reworded the spec sentence accordingly; no cycle
  detection built. [Tests: `extends_of_an_undefined_protocol_raises`,
  `a_forward_extends_reference_is_used_before_defined`.]
- **Deferred (non-gating):** **member-parameter defaults** — evaluating a member's
  *non-first* parameter default during dispatch binding (an edge; needs the driven-default
  machinery). Until then dispatch requires every ordinary parameter supplied. The
  dispatcher-value `is` kind uses the first declarer's kind (a name declared by protocols
  of differing kinds is inherently ambiguous).
- **Goal.** `protocol`/`implement` with real single dispatch.
- **Lands.** Load-time registration of `implement P for T` into a
  `(type, member) → callable` table; **dispatcher cells** (a member name binds a
  `CellKind::Dispatcher`); **single dispatch** on the first argument's runtime
  type as an **ordinary driven call** (can suspend/raise/step); **defaults**
  (fall back to the protocol's default body; error only on an unimplemented
  *required* member); **`extends`** requirement + default inheritance;
  **`P.member(args)`** qualified call always available; **not-implemented** and
  **ambiguity** errors (L§10.3); **S-31** signature conformance + which argument
  is "first" under keyword binding.
- **Spec-delta.** D-M5-4 — **RESOLVED 2026-08-27, spec landed** (App C S-61:
  chain rules in L§10.1–10.3); S-31 — **RESOLVED 2026-08-27, spec landed** (bind first,
  dispatch on the first declared parameter's value; first param no default;
  conformance = arity + block param; no implementation defaults; App C S-31).
- **Depends.** M5.0 (dispatcher cells). Independent of the M5.1–M5.4 module
  track — the parallelizable second person's work.
- **Tests.** dispatch picks the impl by first-arg type; a default is used when
  unimplemented; a required-but-unimplemented member raises naming the gap; an
  ambiguous unqualified call errors, `P.member` disambiguates; `extends` parent
  requirements enforced.

### M5.6 — `is` with protocol values `[S]`
- **Goal.** The protocol case of `is` (M4 carve-out).
- **Lands.** `x is P` ⇔ `x`'s type implements `P` (consults the M5.5 registry),
  replacing `is_op`'s "protocol values arrive at M5" stub.
- **Depends.** M5.5.
- **Tests.** exit criterion #6 (protocol-`is` half); `x is P` true iff `x`'s type
  implements every required member of `P`.

### M5.7 — Real Stringable/Hashable dispatch `[M]` — **DONE**
Native-registered `Stringable`/`Hashable` protocols with native defaults (D-M5-1
above). Interpolation drives an explicit `implement Stringable` (via `enter_unary`
+ `StrInterpRendered`); dict insert/lookup/literal drive an explicit `implement
Hashable` (via `hash_plan` + the `DictBuildHashed`/`IndexReadHashed`/
`IndexAssignHashed` continuations, GC-rooting the in-flight dict). `x is
Stringable`/`is Hashable` reflect native coverage. `print`/error-rendering stay
native. Scalars final; compound render/hash and record defaults deferred to M9a.
Tests in `tests/wellknown.rs` (16). Native + wasm + conformance (180) + hygiene
green.
- **Goal.** Retire the placeholder render/hash seams.
- **Lands.** `stringify::render` → dispatch through the **`Stringable`**
  dispatcher (a driven call; **keep the direct hidden-binding call site** so
  interpolation stays immune to shadowing `to_string`, S-37); dict `hash` seam →
  the **`Hashable`** protocol (preserving coherence: `a == b ⇒ hash(a)==hash(b)`,
  one NaN hash, `-0.0`/`0`/`0.0` equal); built-in implementations enough to green
  interpolation + hashing.
- **Spec-delta.** ★ D-M5-1 (M5/M9a split).
- **Depends.** M5.5.
- **Tests.** exit criterion #6 (interpolation over real dispatch); the M4 string
  and dict determinism gates stay green through the new dispatch path.
- **Risk.** `render` is synchronous today; real dispatch is a driven call that
  can suspend/raise. Interpolation already drives per `{expr}`, but `print` and
  host-raised-value rendering call `render` synchronously — those become
  drive-aware or route through the driven dispatcher. `to_string` that
  suspends/raises mid-render (esp. during *error* rendering) is the nasty corner.

### M5.8 — Prelude as a host-configured star-import `[M]`
- **Goal.** Replace the S-43 seeding with a real implicit prelude import.
- **Lands.** An implicit prelude **wildcard-import** of native module(s), with
  **no program-observable change** — same names, read-only, shadowed the same
  way; the type-value + `Error` shadowing preserved; the linear-scan shadowing
  semantics preserved. **Precedence (user, 2026-08-28; L§11.2/§11.4, App C
  S-13):** the prelude is an ordinary wildcard — no special tier; a
  prelude/user-wildcard distinct-binding collision is `ambiguous-import` at
  use (naming "prelude"); same-cell aliases are one binding (dedup by cell
  identity if the M5.2c mark is by name).
- **Spec-delta.** D-M5-6 — **RESOLVED 2026-08-28** (L§5.1 landed; the warning
  itself is the tracked post-M5.8 follow-up, due before M6).
- **Depends.** M5.2 (wildcard), M5.4 (native modules).
- **Tests.** every `expect-out` fixture written against the seeded `print`
  passes unchanged (the "no observable change" gate); a user `let print = 5`
  shadows the prelude `print`.

### M5.9 — Doodle turtle wrapper + 3-module integration `[M]`
- **Goal.** The cell-aliasing acceptance program.
- **Lands.** A Doodle-side turtle wrapper module declaring `parameter pen_color`
  over the native turtle primitives (the **S-44** shape); the **three-module**
  program proving `with pen_color = …` in user code changes drawing done
  *inside* the imported wrapper.
- **Depends.** M5.2, M5.4, M5.8.
- **Tests.** exit criterion #1.

### M5.10 — Exit review + carve-out green + conformance `[M]`
- **Goal.** The milestone gate.
- **Lands.** All six M5 accept clauses green on **native + wasm**; L§10–§11 +
  §4.11–§4.12 conformance chapters; the determinism gate over module loading +
  dispatch; a multi-lens adversarial exit review (module load × suspend/resume,
  cell-aliasing lifetime/GC, dispatch ambiguity, prelude bootstrap); discharge
  the M5 App C entries (S-8/S-13/S-14/S-31/S-32/S-39/S-44).
- **Depends.** all above.

## Notes on ordering and risk

- **Critical path.** M5.0 → M5.1 → M5.2 → M5.8 → M5.9 → M5.10. **Parallel:**
  M5.5 (protocols) runs against M5.0's dispatcher-cell infra alongside the
  M5.1–M5.4 module track; M5.3 and M5.4 parallelize behind M5.1; M5.6/M5.7 follow
  M5.5. Sensible serial order: **5.0, 5.1, 5.2, 5.5, 5.4, 5.3, 5.6, 5.7, 5.8,
  5.9, 5.10.**
- **Risk peak — M5.0 (the refactor).** `resolved`/`namespace` are instance
  singletons threaded through `step.rs`/`control.rs`/`frame.rs`/`machine.rs`/
  `load.rs`/`gc.rs`; frames carry no `ModuleId`; GC roots one namespace;
  `CellObj` has no kind/provenance. This is the load-bearing chunk.
- **Risk — suspendable module load.** An `import` executed mid-drive resolves →
  parses → **drives** the imported top level to completion, nested arbitrarily
  deep, able to **suspend** with the importer parked, with **cycle detection**
  correct across the suspend. It runs on the engine heap stack (ordinary frames),
  so it is *not* the E§7.6 `NestedSuspend` case — but the loading-set/`loading`
  bookkeeping surviving suspend/resume is the subtle surface; the determinism
  gate must cover it.
- **Risk — two ambiguity sources.** A member name from two wildcard-imported
  protocols is both an S-13 wildcard collision (binding) and an L§10.3 dispatch
  ambiguity (protocol). Decide one coherent story: does it fire at name-use or at
  dispatch, and how does `P.member` interact with a wildcard-aliased `P`?
- **Risk — render dispatch shape** (see M5.7) and **prelude bootstrap
  invariance** (see M5.8).

## Spec-delta obligations coming due in M5

Resolve each **in the spec** as part of the chunk that ships it (plan §8: edit +
App C entry + conformance/test). The ★ items (D-M5-1…D-M5-6) need a **fresh user
ask** when reached; the resolved deltas (S-39/S-13/S-32/S-8/S-52) are cited, not
re-decided. New deltas expected mid-M5.5 (`extends`, keyword-first-arg dispatch)
and at M5.1 (resolver sync/async, E§6).
