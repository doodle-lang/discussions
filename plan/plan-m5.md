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

1. **★ D-M5-1 · Stringable/Hashable — the M5 vs M9a split.** *Blocks M5.7.* The
   plan reads two ways: M5 = "real Stringable/Hashable **dispatch**"; M9a =
   "well-known protocols with **built-in + record defaults** (replacing the M4
   placeholders)." **Recommend (a):** M5 lands the **dispatch machinery** +
   protocol-`is` + just enough native built-in `to_string`/`hash` to green the
   carve-outs (scalars final; compound render text stays the provisional
   `<list>`/`<record>` placeholders); **M9a** lands the Doodle-written defaults
   (record-default `to_string`/`hash`, compound rendering). Matches M9a's "the
   M4/M5 carve-out clauses all green."
2. **★ D-M5-2 · Can the host module resolver suspend?** *Blocks M5.1.* E§6 states
   `resolve(module_path) → Source | NotFound` **synchronously**, and the demo
   bundles all source. A browser host fetching module source over the network
   would need async. **Recommend:** **synchronous resolve for v0.1** (the host
   pre-bundles/pre-resolves source); a suspending source-fetch is a post-v0.1
   spec-delta. (Distinct from "module top-level *driving* can suspend," which is
   already settled, E§6.) New E§6 delta either way.
3. **★ D-M5-3 · S-44 · May native modules declare `parameter` cells?** *Blocks
   M5.4/M5.9.* The plan **assumes not** — dynamic parameters live in **Doodle
   wrapper modules** over native primitives (the turtle wrapper declares
   `parameter pen_color`). **Recommend:** confirm the assumption (native modules
   export functions/consts/foreign/records, not parameter cells); the M5.9
   turtle wrapper is built on it.
4. **★ D-M5-4 · `extends` semantics depth (L§10.1 under-specified).** *Blocks
   M5.5.* L§10.1 pins `extends` as a *requirement* relationship with parent
   defaults reaching child implementors "through the ordinary default-member
   mechanism," but does not pin multi-level default-resolution order, diamond
   legality, or `extends`-cycle handling. **Recommend:** linear chase to the
   root (nearest override wins); an `extends` cycle is a static error; diamonds
   either forbidden or the linear order pinned — user's call.
5. **★ D-M5-5 · `module … end` block form vs file-level module (S-14 / L§11.1
   provisional).** *Blocks M5.3.* L§11.1 marks the file-level/block-level
   interaction provisional. **Recommend:** a file-level `module Name` renames the
   file's module; nested `module` sub-namespaces deferred past v0.1 unless the
   user wants them now.
6. **★ D-M5-6 · S-43 parked shadowing warning (non-gating).** *M5.8 polish.*
   Should declaring a name that hides a prelude name (`let print = 5` then
   `print("hi")` → NotCallable) fire the L§5.1 shadowing warning? Applies to
   type values and intrinsics alike; needs the front end to know the prelude
   name set (natural once the prelude is an import). **Recommend:** yes, warn —
   but it is non-gating and can land with M5.8 or slip.

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

### M5.0 — Multi-module machine foundation `[L]` (critical)
- **Goal.** Turn the single-module machine into a multi-module one, with the
  current module as `ModuleId(0)`. No new language surface.
- **Lands.** A loaded-module table (`ModuleId → {ResolvedModule, namespace,
  state}`); **per-frame `ModuleId`** so a name read hits the right module's
  namespace/AST; `step`/`control`/`read_ref`/`bind_decl` read the current
  frame's module; GC roots **all** loaded modules' namespaces; `CellObj` gains a
  **kind + provenance** tag (`value/const/parameter/dispatcher/module`, AD5) —
  as a field or a parallel table.
- **Depends.** M4. **Under-scoping this cascades into M5.1/M5.2 — do it fully.**
- **Tests.** All existing conformance + determinism gates stay green with the
  single module reframed as `ModuleId(0)`; a unit test proving two modules'
  namespaces are independently rooted.

### M5.1 — Resolver hook + load state machine + suspendable driving `[L]` (critical)
- **Goal.** Load a second module on first reference, drivably.
- **Lands.** The host `resolve(module_path) → Source(text, canonical_id) |
  NotFound` hook (E§6); the `{unloaded, loading, loaded, failed}` state machine
  with **cycle detection that survives a suspend**; driving an imported module's
  top level as **ordinary engine frames** (not a native reentrant drive);
  **circular-import diagnostic naming the cycle**; `failed` + re-raise (S-8); a
  real heap **module value** exposing name + source location + exported
  names/values + docstring (L§4.11, §13 reflection).
- **Spec-delta.** ★ D-M5-2 (resolver sync/async).
- **Depends.** M5.0.
- **Tests.** load-once singleton (top-level effect runs once); a module raising
  at load → `failed`, re-import re-raises; a 2-module cycle → the naming
  diagnostic; a module whose top-level suspends (a capability at load) resumes
  correctly with the importer parked.

### M5.2 — Import forms + cell aliasing + provenance/ambiguity `[M–L]` (critical)
- **Goal.** All import forms, with correct aliasing and read-only provenance.
- **Lands.** Every `ImportTarget` (`import m`, `import m as x`, `import m.x`,
  `import m.x as y`, `import m.*`) with **S-7** dotted-path resolution **at
  load** (whole path as module first, else member; native name beats source);
  **cell aliasing** (a wildcard aliases the exporter's cells, read-only);
  **S-39** assignment-to-imported-name static error carrying the provenance name;
  **S-13** wildcard collision → ambiguous mark, **error on use** naming both
  modules; explicit/selective imports override a wildcard.
- **Spec-delta.** close S-13 (wildcard) + S-39 (provenance-naming residue).
- **Depends.** M5.0, M5.1.
- **Tests.** exit criteria #2 and #3; each import form binds correctly; a
  `with`-bind of an imported parameter cell reaches the exporter's reads.

### M5.3 — `exports` + `module … end` corners `[S–M]` (parallel behind M5.1)
- **Goal.** The public-surface declaration and the module-block form.
- **Lands.** `exports a, b` restricting the public surface; **S-14** corners
  (multiplicity, an undeclared name in an `exports` list, position); the
  `module … end` block-form semantics.
- **Spec-delta.** ★ D-M5-5 (S-14).
- **Depends.** M5.1.
- **Tests.** a non-exported name is not reachable as `m.x`; an `exports` naming
  an undeclared name is a static error.

### M5.4 — Native modules + missing-primitive failure `[M]` (parallel w/ M5.5)
- **Goal.** Host-provided native modules, consulted before source.
- **Lands.** Native-module registration (a member is a foreign function / const
  / foreign value / record), consulted **before** source lookup (E§5.5, §6);
  **S-32** timing (before first drive); an `import` of a module whose platform
  primitive is absent **fails per E§13**; the S-43 intrinsic path kept working
  (or folded into a native prelude module).
- **Spec-delta.** confirm S-32; ★ D-M5-3 (S-44 — no parameter cells in native
  modules).
- **Depends.** M5.0, M5.1.
- **Tests.** exit criterion #5; a native-module member resolves and calls;
  registration after load is a host-API error.

### M5.5 — Protocols: dispatch, defaults, extends, qualified, ambiguity `[L]` (parallel track; critical for accept)
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
- **Spec-delta.** ★ D-M5-4 (`extends` depth); S-31.
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

### M5.7 — Real Stringable/Hashable dispatch `[M]`
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
  semantics preserved.
- **Spec-delta.** ★ D-M5-6 (shadowing warning, non-gating).
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
