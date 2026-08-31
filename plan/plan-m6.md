# Plan — M6: Full observation surface + IDE debugger `[M]`

The working plan for milestone **M6** (implementation plan §5, the M6 paragraph):
the pull-observation surface of **E§8** completed and made resumable-debuggable,
plus the first browser **debugger**. This is the milestone that turns the engine
from "runs programs" into "a program you can watch, pause, step, and inspect."

Status: **draft, awaiting ratification.** Chunk sizes per implementation-plan
Appendix A.

## The one fact that dominates M6

Unlike M4 (machine layer behind `unimplemented!`) and M5 (a single→multi-module
refactor), **M6 has no deep refactor gating it — the drive scaffolding already
exists.** `drive.rs` already defines the full `Directive` enum
(`RunToCompletion`/`Continue`/`Step`/`StepInto`/`StepOver`/`StepOut`), a
`PauseReason` with `Breakpoint(id)`/`RaiseTrap`/`HostPause`/`SliceEnd`,
`BreakpointId`, and a depth-anchored `should_pause` that already implements the
`Step*` directives. **The variants are unwired shells:** `Continue` returns "don't
pause" (no breakpoint/raise-trap check), there is no breakpoint index, no
host-pause flag, and no raise-trap. So M6 is three kinds of work, none of them a
rewrite:

1. **Fill in the unwired shells** — breakpoints, raise-trap, host pause — in the
   drive loop and the raise machinery.
2. **Complete the pull-observation surface (E§8)** — value inspection for records
   and dicts (today only lists + scalars), per-frame locals + `with` dynamic
   bindings, host-facing tail-elided history, a minimal callable-reflection API,
   an observation-mode config, and auxiliary evaluation (S-22).
3. **Build the browser debugger (doodle-web)** — panels + step controls +
   breakpoint gutter over the new surface, plus the conformance that certifies it.

The risk profile is therefore **breadth, not depth**: many small surfaces that must
each be correct, deterministic, and handle-safe. The two genuinely tricky
mechanisms are **raise-trap-before-unwind** (needs a new *paused-mid-raise* machine
state that resumes the unwind) and **auxiliary evaluation** (a nested `to_string`
drive on a *paused* instance under a saved/restored debug context).

Two **ratified pre-M6 obligations** (tracked in `claude-todo.md`) land first as
M6.0, because the IDE consumes them: the **D-M5-6 shadowing warning** and the
**`Error.details` population**.

## M6 exit criteria (accept, from the M6 paragraph)

- Scripted **debugger-session conformance** asserting exact pause positions and
  stack shapes, including a **directive-semantics matrix**: the *same* program with
  a breakpoint and raise-trap **runs to completion under `RunToCompletion`** but
  **pauses under `Continue`**.
- In the live IDE, **`StepOver` on a tail-recursive `polygon`** stops each iteration
  at **constant depth** with the **tail-iteration badge counting**.
- **Raise-trap shows the pre-unwind stack** (the raising frame intact, before
  propagation).
- **A supervised debug session works end-to-end in the browser.**

Deliverable surface (E§8): breakpoints (span index, pending, per-iteration refire);
`Step`/`StepInto`/`StepOver`/`StepOut` with tail-aware depth; raise-trap before
unwind for **all three** raise sources (`raise`, foreign-raise via `resolve(Raise)`,
engine-raise); host pause; the pure structural inspection API; per-subexpression
observation mode; auxiliary evaluation (S-22). IDE panels: call stack with locals,
dynamic bindings, tail-iteration badges, elided-frame history; watch-it-run mode.

## Decisions (all resolved by the user, 2026-08-29)

**D-M6-1 — Per-subexpression observation mode (E§7.4/§8.8). RESOLVED as S-62
(spec landed, discussions `8eed2e3`).** Option (b)'s engineering shape with option
(a)'s **specification discipline**: the fine safe-point set is defined **by syntactic
form** in E§7.4 — the **completion of every non-leaf subexpression** (operator
applications, calls besides entry/return, field access and index steps,
`if`-expression branch results, interpolation pieces) — *not* "wherever a continuation
exists"; in a CESK machine the two coincide, so the specified set is the cheap set.
**Leaves** (literals, name reads) are **not** safe points: nothing observable lies
between them. At a fine safe point the host sees the completed subexpression's span
(§8.1) **and the value it just produced** (the result register, §8.4) — the
"watch your expression evaluate" primitive the IDE builds on. **Observation-only
(load-bearing):** GC, resource limits, cancellation observation, the step budget, and
slice fuel **stay at statement safe points in every mode** (S-20 extended — a fault
landing at a different instant in fine mode would be a mode-dependent fault, a
determinism-gate diff). The fine-mode set is **part of replay identity**; the
determinism gate runs a fine-mode stepping trace twice. Cost: one mode-gated check per
subexpression completion, paid only in fine mode; coarse pays nothing. *Realized in
M6.6.*

**D-M6-2 — Debugger-session conformance format. RESOLVED: portable drive-script format
now.** M6 builds a **new conformance-runner mode** consuming a **drive-script fixture**
(directives + capability/import resolutions in → expected outcome / position / stack
stream out, §4.3), certified through the public surfaces and binding-portable — not
deferred to M8. The M8 record/replay stream format (CBOR + divergence detector) is a
*sibling* artifact; M6's drive-script format is the debugger-session vehicle. *Shapes
M6.8 (now a full chunk, not integration tests).*

**D-M6-3 — IDE debugger scope (doodle-web). RESOLVED: fuller debugger in-gate.** M6.9
delivers the **full** debugger: breakpoint gutter, step controls, a call-stack panel
with locals + `with` dynamic bindings + tail-iteration badges, **plus** watch-it-run
mode, **elided-frame history visualization**, **expandable value trees**, and a
**raise-trap UI**. The environment track (§7.3) then hardens UX on top. *Enlarges
M6.9.*

**D-M6-4 — Callable-reflection engine API (M6) vs the Doodle `help` stdlib (M9a).**
E§8.2 says a frame's callable exposes reflection (name, procedure/function kind,
source location, docstring) "per L§13"; the *Doodle-level* reflection stdlib
(`help`/`type_of`/…) is M9a (§3.8). Provisional (obvious, reversible, **stated not
asked**): expose a **minimal engine-level** callable-reflection API in M6 — a
callable handle's name, `to`/`fn` kind, decl span, and docstring span — which is what
the stack panel needs; the Doodle `help` stdlib stays M9a. Flagged here for veto.

**Also settled (no decision):** live edit / hot redefinition (E§8.9) is **out of
scope** (implementation-plan §1.2 non-goal; Appendix B deferred). Conditional
breakpoints (a Doodle predicate at the safe point) are the noted E§8.6 extension —
**out of scope** for M6 (they run Doodle code; the auxiliary-eval machinery M6.7
builds is their eventual foundation).

## The work items

### M6.0 — Pre-M6 obligations (ratified; no decisions) `[M]`

Two ratified follow-ups from M5, done first because the IDE consumes them.

- **M6.0a — Load-diagnostics record (S-63) + D-M5-6 shadowing warning `[M]` — DONE
  (doodle-rust `b3a9eb3`).** The D-M5-6 warning (a declaration hiding a **prelude** name
  warns per **L§5.1**, ratified `5c627fb`) is inherently **load-time** (it needs the
  prelude's exports), and E§3.2 had no channel for warnings on a *successful* load — so
  the user pinned **S-63** (spec `4be2fa7`), which also closes M1.1's three discovered
  deltas (warnings channel, diagnostic schema, diagnostic ordering). Built: the
  instance-scoped, monotonic **load-diagnostics record** (`Machine.load_diagnostics`),
  appending every front-end diagnostic for every module loaded or attempted — the entry
  module's prelude-shadowing at `load_full`; each imported module's parse+resolve
  diagnostics (previously dropped) plus its prelude-shadowing in `import.rs` — **errors
  included**; the pull accessor `Instance::load_diagnostics(since)`; the D-M5-6 pass
  `load::prelude_shadowing` (globals∩prelude-names diff, no resolver-API change);
  deterministic order (load order, then span order), replay-stable, engine-owned.
  Errors keep their control-flow channels (`LoadError`; a broken import's
  `module-load-error`); the record is the one display surface. `machine.rs` crossed the
  500-line soft limit with the new field, so its six `#[cfg(test)]` `Instance` helpers
  moved to the length-exempt `machine/tests.rs`. Tests `tests/load_diagnostics.rs` (7);
  gates native 519 / conformance 191 / wasm32 / hygiene 6/6. **Deferred (small, M6.9
  facade wiring):** seeding the *entry* module's host-resolve **lexical** diagnostics
  into the record — the host holds them from its own `resolve()`, and the display
  merge is natural where the facade calls `load`.

- **M6.0b — `Error.details` population `[M]`.** Populate the per-kind `details` dict
  for **every** raising `ExceptionKind` (today `make_error` builds `{}` uniformly).
  The **checklist is the rubric App A table** (`error-message-rubric.md`): 15 kinds
  have ratified schemas (`index-out-of-range {index,length}`, `key-not-found {key}`,
  `no-such-field {field,type}`, `module-not-found {path,importer}`, `ambiguous-member
  {member,protocols,type}`, `with-target-not-parameter {name,module,kind}`, …). The
  rubric's remaining rows are **"schema TBD at the rubric pass"** — this chunk
  **proposes** schemas for those (`type-mismatch`, `division-by-zero`,
  `non-finite-float`, `undefined-ordering`, `procedure-in-expression`,
  `not-callable`, `argument-error`, `unhashable-key`, `no-value-destination`,
  `function-fell-off-end`, `host-raised`) as **rubric edits the user ratifies**, then
  populates them. Mechanically: thread structured data from each raise site into
  `make_error`; a `details` builder per kind. Pin **(a) messages-are-not-API** with a
  conformance vector that reads `e.details` (never the message). This touches many
  raise sites — enumerate them **repo-wide** by grepping the `Raise::new`/raise
  helpers, not a guessed subset.

### M6.1 — Structural value inspection: records + dicts + callable reflection `[M]` (foundation) — **DONE (`9be1b18`)**

Complete the **pure** inspection surface (E§4.4/§8.4) — the debugger renders program
state from this, **without running Doodle code**. Today `boundary.rs` inspects only
scalars + lists (`kind_of`, `as_*`, `list_length`/`list_get`). Add:
- **Record inspection** — the record's type name (or a type-handle), its field names
  in declaration order, and each field's value handle.
- **Dict inspection** — length, and key/value handles by **insertion index** (L§4.7
  order), so a host renders entries deterministically.
- **Callable reflection** (D-M6-4, minimal) — for a callable handle: name,
  `to`/`fn` kind, decl span, docstring span. The stack panel's rendering source.
- **Module / type inspection** — enough for the value tree to label a `Module`/`Type`
  value (name + member names for a module).

All handle-minting methods follow the `list_get` discipline (host owns, must
`release`). Tests: round-trip inspection of a nested record/dict/list without
driving; determinism of dict entry order.

### M6.2 — Rich frame observation: locals + dynamic bindings + tail history `[M]`  — **DONE (`d27dadf`)**

Extend the frame surface (E§8.2/§8.3) beyond today's `callable`/`call_site`/
`tail_count`:
- **Local bindings** — per frame, the in-scope parameters and `let`/`const` names as
  **name→value handles**. Source: the frame's environment slots + the resolver's
  slot→name table (needs a slot-name map on `ResolvedModule`/the callable; verify
  what exists, add if missing).
- **Dynamic-parameter bindings** — the `with`-established bindings **in this frame**
  (L§5.5) as name→value handles (from the frame's `dyn_stack`/`dyn_depth`, M4.6).
- **Tail-elided history accessor** — a host-facing read of the ring buffer (already
  captured into traces at raise): each elided frame's callable + call-site, marked
  `tail_elided`, most-recent-first, distinguished from live frames (S-34).

Extends `FrameObservation`; keep the cheap `call_site_spans` path for the live line
highlight. Tests: locals visible at a `Paused` point; a `with` body exposes its
binding; a tail loop shows `tail_count` climbing + a bounded elided history.

### M6.3 — Host-requested pause `[S]`

A pending-pause flag (thread-safe, an atomic polled at safe points — the same
carve-out as cancel, S-23) → `Paused(HostPause)` at the next safe point. Wire into
the drive loop's safe-point check. `RunToCompletion` still honors a host pause? No —
per E§8.8 the host *requests* a pause and the engine stops at the next safe point
**regardless of directive** (it is a host control, like cancel, not a debug stop
gated by the directive). Confirm against E§8.8 wording; test both under `Continue`
and `RunToCompletion`.

### M6.4 — Breakpoints `[M]` (E§8.6, S-21)

- **API** — `set_breakpoint(module_id, line) -> BreakpointId`, `clear_breakpoint(id)`.
- **Span index per loaded module** — line → the first statement-level safe point at
  or after that line. S-21 corners: **lines without code** snap to the next safe
  point; **multiple statements per line** resolve to the first; **pending
  breakpoints** for a not-yet-loaded module are held and installed when it loads;
  **canonical-id reuse** (a re-loaded module) re-resolves.
- **Drive wiring** — under `Continue` or `Step*`, stop `Paused(Breakpoint(id))` at
  the first safe point at/after a breakpointed position; **`RunToCompletion` ignores
  breakpoints**; **per-iteration refire** (a breakpoint inside a loop fires each
  iteration — falls out of "check at each safe point," but test it explicitly).
- This is the first real content of the `Continue` directive (today a no-pause run).

### M6.5 — Raise-trap `[M–L]` (E§8.7, S-18) — *the tricky mechanism*

When raise-trapping is enabled, stop `Paused(RaiseTrap)` at the point an exception is
raised, **before the stack unwinds**, so the debugger inspects the raising frame with
the stack intact; **resuming continues the unwind**. Independent of whether the
exception is later caught. **Unified across all three raise sources** (S-18):
program `raise`, a foreign/host `resolve(Raise)`, and an engine-generated raise — one
"raise at frame F" trap point.

The mechanism: a **paused-mid-raise** state. Today a raise enters the unwind channel
(`Halt::Raise` / the M4.5 unwind) immediately. Raise-trap must (1) capture the trace
(already done at the raise site), (2) pause with frames intact, (3) on the next drive,
**enter the unwind** (run cleanup, search handlers) exactly as if the trap had not
fired. Needs a machine field holding the pending raise across the pause. **Filed as a
spec-delta note if E§8.7 needs sharpening** on the resume contract (it says "resuming
continues the unwind" — confirm that a `Step*`/`Continue` directive on resume is
honored). Test: raise-trap under `Continue` pauses at the raise with the pre-unwind
stack; resume propagates normally; the same program under `RunToCompletion` runs
straight through (the directive-semantics matrix).

### M6.6 — Stepping refinement + observation mode `[M]` (S-62)

- **Tail-aware `StepOver`/`StepOut`** (E§8.5) — a tail call reuses the frame (depth
  unchanged), so the current depth-anchored `should_pause` already keeps a
  tail-recursive loop from running away; **verify** via frame identity (not just
  depth) that `StepOver` stops each `polygon` iteration at constant depth, and add the
  conformance. Refine only if the depth model misbehaves across a tail reuse.
- **Observation-mode config** (E§8.8) — add the mode to `Config`
  (per-statement | per-subexpression; eager vs lazy local capture) and honor S-20.
- **Fine safe points (S-62)** — a **mode-gated check at the completion of every
  non-leaf subexpression** (operator applications; calls, besides their existing
  entry/return; field access + index steps; `if`-expression branch results;
  interpolation pieces). **Leaves are not stopped at.** At a fine safe point expose the
  completed span (§8.1) + the just-produced value from the result register (§8.4). The
  gate is where the drive may return `Paused(Step)`/`Paused(HostPause)` **only** —
  **no** accounting, GC, limit, cancel, budget, or fuel check moves there (they stay at
  statement safe points, so a fault lands at the same instant in both modes). Coarse
  mode pays nothing. Fixtures: a fine-mode stepping trace over operators / calls /
  field+index / if-expr / interpolation; leaves not stopped at; **the same fault
  instant in both modes**; and a **fine-mode determinism double-run** (the set is part
  of replay identity).

### M6.7 — Auxiliary evaluation `[M]` (S-22, E§8.4) — *the tricky mechanism*

Host-driven `to_string` (L§15) on a value at a **paused** instance: a nested drive
that runs Doodle code (may raise/fault) under a **saved/restored debug context** —
its **own small budget**, **breakpoints and raise-trap suppressed**, the outer
directive/pending-pause/step-anchor saved and restored, the paused position
unperturbed on return. Reuses the M2b reentrant-drive machinery; the guardrails
(suppress debug stops, separate budget, restore) are the new part. This is also the
foundation the (deferred) conditional-breakpoint extension would build on. Test:
`to_string` on a record at a breakpoint returns its rendering and leaves the pause
position/stack unchanged; a `to_string` that itself raises surfaces as an
aux-eval error without disturbing the paused program.

### M6.8 — Debugger-session conformance: the drive-script runner `[M–L]` (D-M6-2)

Build the **portable drive-script conformance format** (a new runner mode, §4.3): a
fixture is a **program + a script of directives and capability/import resolutions**;
the expected output is the **stream of outcomes, positions, and stack shapes**. Run it
through the public surfaces so a second implementation is certifiable unchanged. The
scripted assertions the accept names, expressed as drive-script fixtures: exact pause
positions + stack shapes; the **directive-semantics matrix** (breakpoint + raise-trap:
`RunToCompletion` completes, `Continue` pauses); **`StepOver` on tail-recursive
`polygon`** at constant depth with the badge; **raise-trap pre-unwind stack**;
**fine-mode subexpression stepping** (S-62). Includes the determinism gate extended
over the observation surface — stepping / breakpoints / host pause / inspection /
fine mode must not perturb the program-observable trace (E§7.7/§11). The format is a
sibling of the M8 record/replay stream (which adds CBOR + a divergence detector); this
one is the debugger-session vehicle.

### M6.9 — IDE debugger (doodle-web) `[L]` (D-M6-3, fuller in-gate)

The **full** debugger over the new engine surface: a breakpoint **gutter** in the
CodeMirror editor; **step controls** (into/over/out/continue/stop); a **call-stack
panel** with per-frame **locals** + **`with` dynamic bindings** + **tail-iteration
badges**; **elided-frame history visualization**; **expandable value trees** (over the
M6.1 structural inspection); a **raise-trap UI** (pre-unwind stack on a trapped raise);
and **watch-it-run mode** (fine-mode subexpression highlight + the just-produced value,
the S-62 primitive). The pump (M3.5) already fuel-slices; this adds the debug
directives + observation reads + a stopped-UI mode. End-to-end acceptance: set a
breakpoint, run, hit it, inspect locals + expand a record, step, watch an expression
evaluate, resume — in the browser. UX hardening beyond this rides the environment track
(§7.3).

### M6.10 — Exit review + close `[M]`

Multi-lens adversarial **read-only** review (raise-trap × unwind/cleanup interaction;
breakpoint index across module load/canonical-reuse/suspend-resume; handle discipline
across the new minting methods; aux-eval context save/restore; determinism of the
whole surface). App C discharge: **S-15/S-16** (nested-drive faults — verify the
observation surface is consistent under `NestedSuspend`), **S-18** (raise-trap
unification), **S-21** (breakpoint mapping), **S-22** (aux eval), **S-34**
(ring-buffer scoping). Determinism gate green; all four surfaces (native, wasm build,
+ the debugger) green.

## Notes on ordering and risk

- **Critical path:** M6.0 → {M6.1, M6.2} (inspection foundation) → {M6.3, M6.4,
  M6.5} (the three pause sources) → M6.6 → M6.7 → M6.8 (conformance) → M6.9 (IDE) →
  M6.10 (close). M6.1/M6.2 are parallelizable; the three pause sources are
  independent of each other; M6.9 (doodle-web) can begin as soon as M6.1–M6.5 land.
- **The two risk mechanisms are M6.5 (raise-trap paused-mid-raise) and M6.7
  (aux-eval context save/restore).** Both touch the unwind/reentrant-drive machinery
  the R2 tarpit warns about; both get dedicated adversarial attention at M6.10.
- **Breadth risk:** many small handle-minting inspection methods — each must be
  stale-handle-safe and deterministic. The M6.10 handle-discipline lens covers this.
- **Determinism:** stepping, breakpoints, host pause, and inspection are **outside
  replay identity** (E§7.7/§11) — the observation surface must not perturb the
  program-observable trace. The M6.8 gate asserts a debugged run and a straight run
  produce identical program-observable execution.

## Spec-delta obligations coming due in M6

Per implementation-plan §8, App C items shipped by M6 resolve in the spec by M6's
close: **S-18** (raise-trap unification), **S-21** (breakpoint mapping), **S-22**
(auxiliary evaluation), **S-34** (ring-buffer scoping), and any **S-16** clarification
the observation surface forces. Already landed ahead of the code: **S-62** (D-M6-1
fine observation mode — E§7.4/§8.8/App B.1, discussions `8eed2e3`); M6.6 implements it.
Any **M6.5** raise-trap-resume sharpening lands as an E§8.7 edit. The **M6.0b**
rubric-schema proposals land as `error-message-rubric.md` App A edits (the "TBD" rows
filled).
