# CLAUDE.md

Guidance for Claude Code (claude.ai/code) working in this repository.

## What this repository is

SIBYL (Sibylline Strategy Router) — a classifier-and-strategy-lookup service for LLM interaction. See `readme.md` for the public description and `docs/design.md` for the full design specification.

**Current status:** design-only. No runtime code exists yet. The implementation plan targeted at v0.1 should be drafted via `superpowers:writing-plans` before touching code.

## Relationship to sibling projects

SIBYL is part of a cluster of research projects that share vocabulary and infrastructure:

- **convergence-investigation** — SIBYL's strategy dataset is authored by convergence's 7-sub-method research pipeline. Vocabulary (S-class taxonomy, tension-tier scale, apophenia audit) is inherited, not re-invented. When in doubt, defer to convergence's `references/` modules as canon.
- **G0DM0D3** — SIBYL integrates as a service under `G0DM0D3-main/HF/api/`. Reuses `raceModels`, `hf-publisher`, `hf-reader`, `metadata.ts` primitives. Integration-surface file layout is specified in `docs/design.md` §3.1.
- **L1B3RT4S / CL4R1T4S** — input corpora for the initial convergence bootstrap run. Not runtime dependencies.

## Design invariants (from `docs/design.md`)

Any edit to the design or future implementation MUST preserve these invariants:

- **S-class as string, not enum at persistence layer.** Taxonomy evolves without migration.
- **`S-UNKNOWN` pressure-relief valve.** Classifier MUST be able to explicitly return "don't know" to trigger miss-log. Forcing a wrong class masks coverage gaps.
- **Tension thresholds (0.25 / 0.50 / 0.75) match convergence exactly.** Drift from these values is a silent-incompatibility bug.
- **Fail loud on malformed juror output.** No silent salvage, no default-class fallback. Drop invalid votes; if insufficient valid votes remain, return `S-UNKNOWN` with `tension_raw=1.0`.
- **Privacy: hash-only miss-log.** Never persist raw user prompt text server-side. sha256 before logging.
- **Apophenia audit gates publish.** `publish-sibyl.py` MUST refuse to upload if any of the 5 convergence audit checks fail.
- **v2.0 features (auto-routing, scheduled convergence, parseltongue-in-classifier) are explicitly out of scope.** Do not creep these into v0.x–v1.0.

## Operations

- No build system yet. No test runner yet. No runtime dependencies declared yet.
- When implementation begins, follow the structure in `docs/design.md` §3.1 and §9.
- Single integration test file at project root: `test.js`. 200-line max. No scattered test files.
- Implementation follows gm discipline: no comments, no duplication, 200-line file max, fail-loud error handling, witnessed-execution ground truth.

## Before editing

1. Read `docs/design.md` in full. The design is load-bearing for every decision.
2. Check `CHANGELOG.md` for unreleased changes you might be duplicating.
3. If a new unknown surfaces, invoke `gm:planning` rather than patching around it.

## Editing conventions

- Markdown files use the same close-packed prose style as convergence's reference modules — these are read by models at runtime, verbosity costs tokens.
- When extending the design, update both `docs/design.md` and the Status-table in `readme.md` together. They must agree.
- No case-study-specific names, no identifiable individuals, no live-incident details in any committed file. Generic placeholders only (same discipline as convergence).
