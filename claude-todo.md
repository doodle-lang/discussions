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

- [ ] **Ratify `plan/plan-m0.md` and `plan/plan-m1.md`** (both marked
      "submitted for ratification"): ratification pre-approves their
      stated S-item resolutions and scheduled CI wiring. Until then, a
      session may only execute items that need no spec edit or CI wiring.
- [ ] **Accept `plan/machine-design.md` v0.2** — the gate for M2a (and
      its §3 `Value` shape is used by M0.3 scaffolding; see plan-m0).
- [ ] **Repo visibility discrepancy:** §10 D-1 and the original request
      say the repos are private, but workspace/doodle-rust/discussions
      are all currently **PUBLIC** on GitHub. Confirm intended
      visibility; update D-1's text (and D-6/D-7 posture) to match.
- [ ] **Enable GitHub issues** on discussions + doodle-rust (currently
      disabled on all repos; needed by M0.8). Until then,
      implementation.md Appendix C is the spec-delta tracker of record.
- [ ] **S-27 semantic fork** (docstring vs. lone-string body): decide at
      M1.8 start — options + recommendation are written in plan-m1 M1.8.

## In progress

(nothing — next session starts M0.2)

## Next up

Milestone **M0** (see `plan/plan-m0.md` for scope + acceptance):

- [ ] M0.2 — Build/test CI workflow (3-OS test + wasm32 check)
- [ ] M0.3 — Core pipeline skeleton (span/diag/ast/machine/drive shells;
      hard-coded AST runs to Completed)
- [ ] M0.4 — Conformance runner skeleton + format → `conformance/README.md`
      (SKIP-until-implemented pass policy; green CI from day one)
- [ ] M0.5 — wasm hello-world + 300 KB size gate
- [ ] M0.6 — capi hello-world + committed doodle.h + C smoke
- [ ] M0.7 — insta + fuzz plumbing
- [ ] M0.8 — CONTRIBUTING (incl. review policy) + issue templates
      (needs issues enabled — see Awaiting the user)
- [ ] M0.9 — M0 exit review

Then milestone **M1** (see `plan/plan-m1.md`): M1.1 … M1.15, with
conformance tests landing at `stage: lex/parse` per item and upgraded to
`full` at the M1.10 checkpoint.

**Gate before M2a:** `plan/machine-design.md` (v0.2, revised after
adversarial review) must be accepted by the user before any machine-core
code.

## Spec-delta queue (near-term)

Full backlog: `plan/implementation.md` Appendix C (S-1…S-46; the
appendix is the tracker of record until GitHub issues open). Due with M1
work items (resolve in spec *before/with* the implementing item —
pairings in `plan-m1.md`): S-1 (M1.2), S-2 (M1.3), S-3 (M1.5), S-4
(M1.7), S-5 + S-6-in-full (M1.10), S-7 (M1.9), S-11 (M1.10), S-27 (M1.8,
user decision), S-45 (M1.11). Later: S-41 by M2a; **S-9** (machine-design
§12 carries the proposed resolution) by M2a; **S-46** (new: non-local
exits through native consumers) by M2b.

## Open decisions awaiting the user (non-blocking)

From `plan/implementation.md` §10: D-2 (spec home at freeze), D-4 (names/
npm scope), D-5 (Unicode pin verification), D-6 (demo posture), D-7
(privacy/analytics), D-8 (hosting + release cadence). D-1 and D-3 are
resolved (but see the visibility discrepancy above).

## Done

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
