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

## [0.0.0] — 2026-04-23

### Added
- Project initialised
- `readme.md`, `LICENSE` (MIT), `.gitignore`, `CLAUDE.md`
- Design specification committed under `docs/design.md`
