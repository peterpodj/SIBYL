# SIBYL — Sibylline Strategy Router for LLM Interaction

**Date:** 2026-04-23
**Status:** Design approved, pre-implementation
**Name:** SIBYL (Sibylline Strategy Router)
**Scope:** v0.1–v1.0 (v2+ flagged out-of-scope)

---

## 1. What SIBYL Is

SIBYL is a service integrated into G0DM0D3 that:

1. **Classifies** incoming user requests into a structural S-class taxonomy via an LLM **self-jury** (N parallel lightweight jurors), producing a primary class, secondary classes, and a **WFGY-style tension score** over juror disagreement.
2. **Looks up** ranked strategies keyed by `(s_class × tension_tier)` from a **Hugging Face dataset** authored by offline **convergence** investigations.
3. **Logs misses** (low confidence, `S-UNKNOWN`, uncovered classes) into the existing telemetry pipeline. Miss log becomes the input to the next human-initiated convergence re-investigation.
4. Ships as an **opt-in primitive** (`POST /api/analyze`) with an optional advisory UI behind a SettingsModal toggle. Default UX unchanged; power-users opt in.

**Epistemic dialect:** S-class taxonomy, tension-tier scale (Low/Medium/High/Critical), apophenia audit, and versioned knowledge-graph/dataset patterns are all inherited from the convergence methodology (see `convergence-main/convergence-investigation-2.skill/`). SIBYL does not invent a new vocabulary — it re-uses convergence's.

---

## 2. Design Decisions (converged during brainstorming)

| # | Axis | Choice |
|---|---|---|
| Q1 | Form-factor | **Service integrated into G0DM0D3** (not a standalone skill) |
| Q2 | Strategy shape | **Tiered discriminated union**: `prompt_only \| combo \| pipeline` |
| Q3 | Taxonomy | **S-class inspired (WFGY)**, shared with convergence |
| Q4 | Classifier | **LLM self-jury** with asymmetric-self-consistency tension scoring |
| Q5 | Storage | **Hugging Face dataset**, revisions = versioning |
| Q6 | Convergence invocation | **Bootstrap + on-demand + miss-logging** (no auto-triggering) |
| Q7 | Frontend | `/analyze` **backend primitive** + **settings-gated advisory UI** |

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     G0DM0D3 (existing)                          │
│                                                                 │
│   ChatInput ──► /api/chat ──► [GODMODE / ULTRAPLINIAN / ...]    │
│       │                                                         │
│       │  (opt-in advisory panel, settings-gated)                │
│       ▼                                                         │
│   /api/analyze ◄── SIBYL (new) ──► HF Dataset (strategies.v*)   │
│       │                 │                    ▲                  │
│       │                 │                    │                  │
│       │            self-jury              convergence           │
│       │          (5-model race)           (offline,             │
│       │                 │                  on-demand)           │
│       │            tension score                                │
│       ▼                 │                                       │
│   response         ranked strategies                            │
│   { class,           from DB                                    │
│     tension,                                                    │
│     strategies[] }                                              │
│                                                                 │
│   metadata.ts: recordEvent('classification_miss', ...)          │
│                     ▲                                           │
│                     │                                           │
│            (low-confidence or uncovered S-class)                │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1 Component manifest

| Component | Status | Location |
|---|---|---|
| `/api/analyze` route | NEW | `G0DM0D3-main/HF/api/routes/analyze.ts` |
| `lib/sibyl/classifier.ts` (self-jury) | NEW | `G0DM0D3-main/HF/api/lib/sibyl/` |
| `lib/sibyl/tension.ts` | NEW | `G0DM0D3-main/HF/api/lib/sibyl/` |
| `lib/sibyl/strategies-db.ts` (HF reader + cache) | NEW | `G0DM0D3-main/HF/api/lib/sibyl/` |
| `lib/sibyl/s-class.ts` (taxonomy seed) | NEW | `G0DM0D3-main/HF/api/lib/sibyl/` |
| `lib/sibyl/types.ts` | NEW | `G0DM0D3-main/HF/api/lib/sibyl/` |
| `raceModels` | REUSED | `HF/api/lib/ultraplinian.ts:raceModels` |
| `hf-reader.ts` | REUSED (activate unused `flushDataset`) | existing |
| `metadata.ts: recordEvent` | EXTEND with `classification_miss` | existing |
| `SettingsModal.tsx` advisory toggle | EXTEND (new tab) | existing |
| `convergence-investigation-2.skill` | REUSED (manual invocation) | `convergence-main/` |
| `convergence-main/bootstrap-corpus/` | NEW | new directory |
| `convergence-main/scripts/publish-sibyl.py` | NEW | convergence scripts dir |

---

## 4. Data Model

### 4.1 S-Class taxonomy (seed set)

Seed taxonomy ships in `s-class.ts` so v0.1 works before bootstrap finishes. Convergence's bootstrap run may add, remove, or rename classes; the DB schema stores S-class as a **string** (not a TypeScript enum at persistence layer), so the taxonomy can evolve without migration.

```ts
export const S_CLASS_SEED = [
    'S-factual-recall',
    'S-code-generation',
    'S-creative-latitude',
    'S-analytical-reasoning',
    'S-boundary-probe',
    'S-identity-reframe',
    'S-multi-turn-escalation',
    'S-domain-locked',
    'S-adversarial-obfuscated',
    'S-meta-request',
    'S-emotional-support',
    'S-research-inquiry',
    'S-UNKNOWN'
] as const
export type SClass = typeof S_CLASS_SEED[number]
```

`S-UNKNOWN` is the **pressure-relief valve**: classifier returns it explicitly when no seed class fits, triggering a miss-log. Without it, the jury is forced into a wrong class, masking coverage gaps.

### 4.2 Strategy schema (discriminated union)

```ts
export type StrategyTier = 'prompt_only' | 'combo' | 'pipeline'

export interface StrategyBase {
    id: string
    s_class: string
    tension_tier: 'low' | 'medium' | 'high' | 'critical'
    efficacy_score: number
    tier: StrategyTier
    provenance: {
        source: 'L1B3RT4S' | 'CL4R1T4S' | 'GODMODE' | 'parseltongue' | 'convergence-synth' | 'community-pr'
        evidence_refs: string[]
        investigated_at: string
        convergence_version: string
    }
    apophenia_checks: {
        tier_check: boolean
        counter_evidence_searched: boolean
        novelty_confirmed: boolean
    }
}

export interface PromptOnlyStrategy extends StrategyBase {
    tier: 'prompt_only'
    template: string
    variables: string[]
    target_model_families: string[]
}

export interface ComboStrategy extends StrategyBase {
    tier: 'combo'
    model_id: string
    prompt_template: string
    variables: string[]
    sampling: {
        temperature?: number
        top_p?: number
        max_tokens?: number
    }
}

export interface PipelineStrategy extends StrategyBase {
    tier: 'pipeline'
    steps: Array<
        | { kind: 'perturb'; module: 'parseltongue'; technique: string; intensity: number }
        | { kind: 'classify'; module: 'sibyl'; depth: 'primary' | 'recursive' }
        | { kind: 'race'; module: 'ultraplinian'; tier: 'fast' | 'standard' | 'full' }
        | { kind: 'transform'; module: 'stm'; module_id: string }
        | { kind: 'completion'; model_id: string; prompt_template: string }
    >
    final_step: 'race' | 'completion'
}

export type Strategy = PromptOnlyStrategy | ComboStrategy | PipelineStrategy
```

Pipeline `steps` is a declarative composition of **existing G0DM0D3 primitives** (parseltongue, sibyl, ultraplinian, STM, completions). Convergence extends SIBYL's capability by emitting new pipelines — not new runtime code.

### 4.3 Tension score

```ts
export interface TensionScore {
    raw: number
    tier: 'low' | 'medium' | 'high' | 'critical'
    votes: Array<{ model_id: string; s_class: string; confidence: number }>
    agreement_ratio: number
}

export function tensionTier(raw: number): TensionScore['tier'] {
    if (raw < 0.25) return 'low'
    if (raw < 0.50) return 'medium'
    if (raw < 0.75) return 'high'
    return 'critical'
}
```

Thresholds match convergence's `bridge-figures.md` and `epistemic-tiers.md` exactly.

### 4.4 HF dataset row format (JSONL)

```json
{
  "id": "uuid",
  "s_class": "S-boundary-probe",
  "tension_tier": "critical",
  "tier": "combo",
  "efficacy_score": 0.82,
  "payload_json": "{ ...full Strategy ... }",
  "provenance_source": "convergence-synth",
  "convergence_version": "2.0.0",
  "investigated_at": "2026-04-23T12:00:00Z",
  "dataset_rev": "sibyl-v1.0"
}
```

Dataset splits: `strategies/` | `miss_log/` | `s_class_meta/`.

### 4.5 Miss-log event

```ts
{
    event_type: 'classification_miss',
    reason: 'low_confidence' | 'unknown_s_class' | 'high_tension_no_strategy',
    prompt_hash: string,         // sha256(prompt) — NEVER raw prompt
    prompt_length: number,
    classification: {
        primary_s_class: string,
        tension_raw: number,
        confidence: number,
        votes_count: number
    },
    available_strategy_count: number,
    user_tier: string,
    timestamp: number
}
```

**Privacy invariant:** prompt text is hashed before logging. Never persist raw user prompts server-side.

---

## 5. Classifier (self-jury)

### 5.1 Jury composition

- **5 jurors** by default (odd for unambiguous plurality)
- Drawn from ULTRAPLINIAN's FAST tier
- Configurable via env:

```
SIBYL_JURY_MODELS=anthropic/claude-haiku-4-5,google/gemini-2.5-flash,meta-llama/llama-3-8b,mistralai/mistral-7b,qwen/qwen-2.5-7b
SIBYL_JURY_SIZE=5
SIBYL_CONFIDENCE_THRESHOLD=0.6
SIBYL_TENSION_ESCALATION=0.50
SIBYL_CACHE_TTL_MS=300000
SIBYL_DATASET_REV=   # empty = track HEAD
```

### 5.2 Juror prompt (reused across all jurors for prompt-cache hits)

```
SYSTEM:
You are a classifier. Read the user's request below and assign it to exactly one
S-class from the taxonomy. Return ONLY a JSON object:

  { "s_class": "<one of: S-factual-recall | ... | S-UNKNOWN>",
    "secondary": [ ...up to 2 other plausible S-classes ],
    "confidence": <float 0-1>,
    "reasoning": "<one sentence, for audit>" }

S-class definitions:
<inlined ~400 tokens — identical across jurors, cache-friendly>

USER:
<request text, truncated to 4KB>
```

### 5.3 Tension aggregation formula

```
margin        = (winner_votes - runner_up_votes) / total_votes
disagreement  = 1 - margin
uncertainty   = 1 - (winner_votes_confidence_avg)
tension_raw   = 0.6 × disagreement + 0.4 × uncertainty
```

**Note:** this is a **structural analog** of convergence's WFGY asymmetric self-consistency, not a 1:1 port. Convergence's formula is defined for bridge-figure role/context tension; SIBYL re-applies the 0.00–1.00 scale and 4-tier interpretation to classifier voting. If convergence ships a general-purpose tension module, SIBYL swaps implementations at the `tension.ts` boundary.

**0.6 / 0.4 weights are first-pass heuristics** (mutable #1 below).

### 5.4 Fail-safe: malformed juror output

- Non-JSON or missing-field votes are **dropped** (not retried)
- If valid votes < `ceil(JURY_SIZE / 2)` → return `S-UNKNOWN` with `tension_raw=1.0` + immediate miss-log
- **No silent salvage, no default class, no retries.** Fail loud.

### 5.5 Parallelism

Reuses `raceModels` from `HF/api/lib/ultraplinian.ts` with a classify-payload. No new concurrency primitive.

---

## 6. Strategies DB on HF

### 6.1 HF dataset layout

```
HF_DATASET_REPO/
├── strategies/{convergence_version}.parquet    # one row per Strategy
├── miss_log/{YYYY-MM}.parquet                  # append-only, service-written
├── s_class_meta/{convergence_version}.parquet  # taxonomy snapshot
└── README.md                                   # auto-generated coverage summary
```

Revision model: **HF git revisions**. Service reads HEAD by default; pinnable via `SIBYL_DATASET_REV`.

### 6.2 Read path

In-process cache + revision polling:

- Full dataset in memory (expected <10k rows × ~2KB = ~20MB)
- TTL poll every 5 min checks HF HEAD revision; reload only if changed
- Manual invalidation endpoint: `POST /api/admin/sibyl/reload` (tierGate-protected)

### 6.3 Write path (convergence → HF)

- `convergence-main/scripts/publish-sibyl.py` uploads `outputs/sibyl-v{N.M}/` to HF
- **Apophenia audit gate:** publish refuses if any of the 5 audit checks failed
- Reuses `huggingface_hub` (already a dep via OBLITERATUS-main)

### 6.4 Miss-log write path

Reuses existing `hf-publisher.ts` batching pipeline (`HF_FLUSH_THRESHOLD`, `HF_FLUSH_INTERVAL_MS`). New event type, same pipe. Zero new infra.

### 6.5 Query patterns

| Query | Runtime | Implementation |
|---|---|---|
| `lookup(s_class, tension_tier)` | O(1) | Map lookup in cache |
| `coverage()` | O(n) | In-memory aggregation |
| `missLog(since)` | Network | HF parquet read |
| `byId(id)` | O(n) | Cache scan (acceptable — admin/audit path only) |

---

## 7. Bootstrap via Convergence

### 7.1 Corpus

```
convergence-main/bootstrap-corpus/
├── l1b3rt4s/              # copy of L1B3RT4S-main/*.mkd
├── cl4r1t4s/              # copy of CL4R1T4S-main/*
├── g0dm0d3/               # godmode-prompt.ts, parseltongue.ts, ultraplinian configs
├── external/              # URLs for AdvBench, HarmBench, etc.
└── MANIFEST.md            # maps each file to expected S-classes
```

Bootstrap is **local-first**: nothing leaves the user's machine except explicit URL fetches.

### 7.2 Convergence sub-methods mapped to bootstrap

| Sub-method | Application |
|---|---|
| 1. Structural-Parallel Documenter | Cross-vendor refusal-surface matrix |
| 2. Bridge-Figure Mapper | Cross-S-class prompts (high-value bridges) |
| 3. Document Parser | Extract prompts/params from .mkd files |
| 4. Corporate-Jurisdiction Verifier | **Reframed:** model-provider registry verifier (OpenRouter model ID validation) |
| 5. Graph Integration | Strategy graph: nodes=strategies, edges=(same s_class, same perturbation, same corpus) |
| 6. Handoff Generation | End-of-run delivery doc |
| 7. Visualization Artifacts | D3 atlas: s_class × tension_tier × strategy count |

Sub-method 4 is a deliberate semantic reframe (model-provider ≈ corporate registry). Documenting this explicitly to prevent future mechanical drift from convergence canon.

### 7.3 Per-S-class investigation unit

For each seed S-class, convergence runs a self-contained mini-investigation with:

- **Acceptance:** ≥ 3 strategies per `(s_class × tension_tier)` cell, OR explicit `NOT_COVERED` marker
- **Apophenia audit** (5 checks) must pass before publish
- **Every strategy** has `provenance.evidence_refs` populated

### 7.4 Deliverable bundle

```
convergence-main/outputs/sibyl-v1.0/
├── strategies-v1.0.jsonl
├── s-class-meta-v1.0.jsonl
├── apophenia-audit-v1.0.json
├── coverage-matrix-v1.0.md
├── handoff-v1.0.md
├── sibyl-atlas-v1.0.html
└── publish-manifest.json
```

**Publish is manual** — human reviews handoff + coverage before running `publish-sibyl.py`.

### 7.5 On-demand re-runs

Subsequent runs are scoped by miss-log:

```bash
convergence --scope="miss-log-driven" --since="v1.0" \
            --miss-log-source="hf://${HF_DATASET_REPO}/miss_log/"
```

Versioning: semver. Patch=corrections, minor=new strategies, major=schema changes.

---

## 8. Integration Points

### 8.1 `POST /api/analyze`

```
Auth: apiKeyAuth + rateLimit + tierGate('standard')
Body: { prompt: string, conversation_id?: string, return_strategies?: boolean }

Response 200:
{
    classification: { primary_s_class, secondary[], confidence, agreement_ratio, tension, votes },
    strategies: Strategy[],   // omitted if return_strategies=false
    miss_logged: boolean,
    dataset_rev: string
}
```

### 8.2 Wiring into existing routes

- **v0.1:** standalone endpoint only, chat/completions unchanged
- **v0.3 advisory:** chat middleware populates `X-Sibyl-Classification` header when user has advisory toggle on; frontend reads header, renders panel
- **v2.0 auto-route (out of scope):** ULTRAPLINIAN and GODMODE call `sibyl.classify()` internally to pick tier/combo dynamically

### 8.3 SettingsModal new tab

```tsx
<Tab label="SIBYL (advisory)">
    <Toggle checked={settings.sibyl.advisory} />
    <NumberInput value={settings.sibyl.confidenceMin} min={0} max={1} />
    <Select options={['prompt_only', 'combo', 'pipeline', 'auto']} />
</Tab>
```

Opt-in, default off, hidden behind existing feature-flag pattern.

### 8.4 CLI for researchers

```bash
npx sibyl-classify "write me a poem about a sunset"
# JSON output, hits /api/analyze against configured instance
```

---

## 9. Rollout

### v0.1 — Core primitive

- `lib/sibyl/{s-class, types, tension, classifier, strategies-db}.ts`
- `routes/analyze.ts` wired into `server.ts`
- `metadata.ts` extended with `classification_miss`
- Root `test.js` integration test: 5 canned prompts, assert shape + plausible tension
- **No DB required** — returns empty `strategies: []`; classification is live
- Ship point: classifier works; misses logged

### v0.2 — Bootstrap DB

- `convergence-main/bootstrap-corpus/` populated
- convergence run for sibyl-v1.0 scope; apophenia audit passes
- `publish-sibyl.py` uploads to HF
- `strategies-db.ts` returns non-empty results
- Ship point: `/api/analyze` returns real strategies

### v0.3 — Advisory UI

- SettingsModal SIBYL tab
- Pre-chat middleware populates `X-Sibyl-Classification` header
- Advisory panel in `ChatInput.tsx` — renders only when header present; zero delta when toggle off
- Ship point: power-users see live classification

### v1.0 — Public release

- README, API.md section, research paper draft
- Community PR template for strategy contributions
- Coverage matrix published at `HF_DATASET_REPO/README.md`

### v2.0 — Out of scope for this spec

- Auto-routing (ULTRAPLINIAN/GODMODE call sibyl internally)
- Scheduled re-investigation (cron)
- Parseltongue-integrated perturb-then-classify

### Testing policy

- **One `test.js` at project root** (gm discipline)
- Classifier: 15 canned prompts across S-classes, primary class correct ≥70%, plausible tension tier
- Strategies DB: round-trip mock HF dataset through reader, assert sorted lookup
- Miss-log: `S-UNKNOWN` prompt → event in local buffer before flush
- Apophenia audit: malformed audit file → publish-sibyl.py exits non-zero

---

## 10. Open Mutables (resolve during implementation)

| # | Mutable | Where resolved |
|---|---|---|
| 1 | Tension-formula weights (0.6 / 0.4) | v0.2 — calibrate against 100-prompt labeled eval |
| 2 | Jury size and model composition | v0.2 — benchmark alternatives |
| 3 | Cache TTL (5 min) | Observe real dataset update frequency |
| 4 | S-class seed set (12 classes) | Convergence bootstrap may expand/collapse |
| 5 | `S-adversarial-obfuscated` × parseltongue interaction | v0.2 — test with live parseltongue outputs |
| 6 | Prompt hashing scheme (sha256) | Revisit if fuzzy miss-clustering becomes useful → LSH |

---

## 11. Non-Goals (explicit scope exclusions)

- **Not a refusal-bypass tool in itself.** SIBYL classifies and routes; it does not generate adversarial content. Strategy DB rows *reference* techniques; actual execution happens in the existing modes (GODMODE, ULTRAPLINIAN).
- **Not a replacement for GODMODE or ULTRAPLINIAN.** Both modes remain as-is. SIBYL is additive.
- **Not a training-data labeler.** Miss-log captures metadata only (hashed), not raw prompts; unsuitable for training dataset construction without additional pipeline work.
- **Not an autonomous self-extension system in v1.** Convergence re-runs are human-initiated. Auto-triggering is v2+.

---

## 12. Dependencies

### Runtime (new/extended)

- `@huggingface/hub` or equivalent reader — pull dataset parquet at boot
- Existing: `express`, `cors`, ULTRAPLINIAN's `raceModels`, `hf-publisher`, `hf-reader`, `metadata.ts`

### Bootstrap (offline)

- `huggingface_hub` (Python) — already installed via OBLITERATUS-main
- `convergence-investigation-2.skill` — manually linked into `~/.claude/skills`

### Env

```
HF_DATASET_REPO=<user-org>/<repo-name>
HF_TOKEN=<write token>
SIBYL_JURY_MODELS=...
SIBYL_JURY_SIZE=5
SIBYL_CONFIDENCE_THRESHOLD=0.6
SIBYL_TENSION_ESCALATION=0.50
SIBYL_CACHE_TTL_MS=300000
SIBYL_DATASET_REV=   # empty = HEAD
```

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| Classifier jury cost per request | Prompt caching (constant SYSTEM block across jurors) + tierGate gating |
| HF dataset read latency | In-process cache, revision-polling, 5-min TTL |
| Taxonomy drift between convergence and SIBYL | S-class stored as string, not enum; dataset `convergence_version` field tracks generation |
| Adversarial classifier evasion | `S-adversarial-obfuscated` class + v0.2 parseltongue-perturbation test suite |
| HF dataset compromise (supply chain) | `SIBYL_DATASET_REV` pinning + apophenia-audit-gated publish |
| Privacy leak via prompt text | Hash-only miss-log; explicit "never raw prompt" invariant |
