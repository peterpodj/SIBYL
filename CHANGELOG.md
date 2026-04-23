# Changelog

All notable changes to SIBYL will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Design specification (`docs/design.md`) — Sibylline Strategy Router for LLM Interaction
- Architecture: classifier self-jury + HF-dataset-backed strategies DB + miss-log feedback
- Component manifest targeting G0DM0D3 integration
- Rollout plan v0.1 → v1.0 with v2.0 explicitly out of scope
- Open-mutable register for tension-formula calibration, jury composition, cache TTL, seed-taxonomy shape
- Staged implementation plans (`docs/plans/v0.1-core/`, `docs/plans/v0.2-bootstrap/`, `docs/plans/v0.3-advisory-ui/`)
  - v0.1: 12 TDD-disciplined tasks for classifier + `/api/analyze` + miss-log
  - v0.2: 7 tasks for corpus bootstrap, convergence runs, and audit-gated HF publish
  - v0.3: 10 tasks for opt-in advisory UI in G0DM0D3 SettingsModal + ChatInput

### Changed
- v0.1 plan amendments after WFGY reasoning-introspection pass exposed unchecked assumptions:
  - Backend target: `G0DM0D3-main/HF/api/` → `G0DM0D3-main/api/` (the full Express backend, with `tierGate`, `hf-publisher`, and `consortium` / `research` routes). `HF/api/` is the reduced HF Spaces deployment variant.
  - `raceModels` wrapper code rewritten to match real signature: `(models: string[], messages: Message[], apiKey: string, params, config?: RaceConfig)`. `ModelResult.model` (not `.model_id`).
  - `metadata.ts` extension: real shape is structured `MetadataEvent` — v0.1 adds an optional `sibyl?:` sub-field (with `classification?` and `miss?` sub-records) and widens the `mode` union with `'analyze'`. Not a tagged-enum append.
  - Task 11 accuracy gate: downgraded from hard-fail at <70% to `console.warn` — 5-prompt sample is noise-limited; calibrated gate deferred to v0.2 per `docs/design.md` §10 mutable #1.
  - Task 12 push/PR: downgraded to local-commit-only + `SIBYL_V0.1_HANDOFF.md` note. Remote-decision (fork `peterpodj/G0DM0D3`, PR upstream, or private vendor fork) is deferred; `G0DM0D3-main` was imported from a zip with no remote.
  - Prerequisites updated to include explicit git-initialisation step for zip-imported working copies.

## [0.0.0] — 2026-04-23

### Added
- Project initialised
- `readme.md`, `LICENSE` (MIT), `.gitignore`, `CLAUDE.md`
- Design specification committed under `docs/design.md`
