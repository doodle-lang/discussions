# Doodle TODO / Status

The living work queue and status snapshot for the Doodle project — the
first thing a fresh session should read after the workspace `CLAUDE.md`.
Keep it current: update when work starts/lands, and record discovered
bugs per the Bug Discovery Protocol (CLAUDE.md). Detail lives in the
working plans (`plan/plan-m*.md`); this file is the index.

Conventions: `[ ]` todo · `[~]` in progress (note the session/branch) ·
`[x]` done (move to the Done log with date+commit). CRITICAL/MAJOR bugs
go at the top, per CLAUDE.md.

## CRITICAL

(none)

## MAJOR

(none)

## Awaiting the user (blocking)

(none — plans ratified, machine-design v0.2 accepted, repos deliberately
public, issues enabled on discussions + doodle-rust: all 2026-07-10.
Still queued for later: the S-27 semantic fork is decided with the user
at M1.8 start — options + recommendation in plan-m1 M1.8.)

## In progress

(nothing — M0.5 landed; M0.6 is next)

## Next up

Milestone **M0** (see `plan/plan-m0.md` for scope + acceptance):

- [ ] M0.6 — capi hello-world + committed doodle.h + C smoke
- [ ] M0.7 — insta + fuzz plumbing
- [ ] M0.8 — CONTRIBUTING (incl. review policy) + issue templates
- [ ] M0.9 — M0 exit review

Then milestone **M1** (see `plan/plan-m1.md`): M1.1 … M1.15, with
conformance tests landing at `stage: lex/parse` per item and upgraded to
`full` at the M1.10 checkpoint.

**M2a gate:** satisfied — `plan/machine-design.md` v0.2 accepted by the
user 2026-07-10. (Mechanism changes still require revising that document
first.)

## Spec-delta queue (near-term)

Full backlog: `plan/implementation.md` Appendix C (S-1…S-46; the
appendix is the tracker of record until GitHub issues open). Due with M1
work items (resolve in spec *before/with* the implementing item —
pairings in `plan-m1.md`): S-1 (M1.2), S-2 (M1.3), S-3 (M1.5), S-4
(M1.7), S-5 + S-6-in-full (M1.10), S-7 (M1.9), S-11 (M1.10), S-27 (M1.8,
user decision), S-45 (M1.11). Later: S-41 by M2a; **S-9** (machine-design
§12 carries the proposed resolution) by M2a; **S-46** (new: non-local
exits through native consumers) by M2b.

Discovered at M0.3 (needs an S-number when the user next curates Appendix
C): **top-level `Completed` value** — E§7.2 pins the `Completed(value?)`
payload only for a returning `fn`; it does not say what a *top-level*
module drive completes with. The M0.3 skeleton provisionally returns the
result register's last value so the acceptance can observe it; real
top-level completion is expected to be Void (`None`). Resolve in E§7.2 by
M2a (when the machine core replaces the placeholder). No behavior ships
before then.

## Open decisions awaiting the user (non-blocking)

From `plan/implementation.md` §10: D-2 (spec home at freeze), D-4 (names/
npm scope), D-5 (Unicode pin verification), D-6 (demo posture), D-7
(privacy/analytics), D-8 (hosting + release cadence). D-1 and D-3 are
resolved (but see the visibility discrepancy above).

## Done

- 2026-07-10 — M0.5: wasm hello-world + size gate. `crates/doodle-wasm`
  (wasm-bindgen cdylib exporting `version()`); `scripts/wasm-size.sh`
  (build release wasm → `wasm-opt -Oz` → brotli → 300 KB budget, plan §6.5;
  fail-closed; `WASM_BUDGET_BYTES`-overridable); a `wasm-size` CI job that
  also self-tests the fail path. Hello-world ≈ 8 KB brotli. All CI +
  hygiene green. 2-lens review: one major (fail-closed) folded in.
  doodle-rust `ad8bbbe`.
- 2026-07-10 — M0.4: conformance test format + runner skeleton.
  `conformance/README.md` (ratified format v0, source of truth); a std-only
  `tools/conformance-runner` (discovers `.doodle` tests, parses/validates
  `#!` directives, SKIPs by doodle-core's `stage::implemented_through()`,
  prints `=== N passed, N failed, N skipped ===`); doodle-core `stage`
  module; four example tests (all SKIP at M0); a `conformance` CI job. All
  CI + hygiene green. 3-lens review: one major (a seed example misfiled
  under L6.2 vs L8.4) + minor robustness/fidelity items, all folded in.
  doodle-rust `2c8ae7e`.
- 2026-07-10 — M0.3: core pipeline skeleton in `doodle-core` — `span`,
  `diag`, `ast`, `machine` (`Value` per machine-design §3; `InstanceState`
  per E§3.3; result register + step cursor), `drive` (`Outcome` per E§7.2;
  `run()` drives a hand-built one-statement program to `Completed`). No
  parser, no machine-core mechanisms (M2a gate). Acceptance test observes
  `Int(42)` through the public API; hygiene + CI green. 3-lens adversarial
  review: no blocker/major; minor findings folded in. Spec-delta filed
  above (top-level `Completed` value). doodle-rust `c60c336`.
- 2026-07-10 — M0.2: build/test CI workflow — `.github/workflows/test.yml`
  with four jobs (`cargo test --workspace` on ubuntu/macos/windows +
  `cargo check --workspace --target wasm32-unknown-unknown` on ubuntu); the
  toolchain (channel + wasm32 target) is provisioned by an argument-free
  `rustup toolchain install` reading `rust-toolchain.toml`. All four jobs
  green on GitHub-hosted runners. doodle-rust `5354c38`.
- 2026-07-10 — Working plans `plan-m0.md`/`plan-m1.md` +
  `machine-design.md` drafted, adversarially reviewed (3 reviewers), and
  revised: machine-design v0.2 (unwinding redesigned around
  resolver-annotated exit targets; S-46 filed; S-9 resolution proposed),
  conformance format v0 gains stage/expect-warning directives, this todo
  file created.
- 2026-07-10 — M0.1: workspace + doodle-rust repos, hygiene checks + CI
  (green), MIT license (D-3), toolchain installed. Commits:
  workspace `a011bb7`, doodle-rust `a7ddf9c`, discussions `1b6d4ce`.
- 2026-07-09 — Implementation plan v0.1 (adversarially reviewed);
  D-1 resolved 07-10.
- 2026-07-04…09 — Language spec v0.1 (incl. string model, dict order),
  engine spec v0.1.
