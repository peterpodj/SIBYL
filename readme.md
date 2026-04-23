# SIBYL

**Sibylline Strategy Router for LLM Interaction.**

SIBYL is a classifier-and-strategy-lookup service for LLM interaction. It reads an incoming user request, consults a panel of lightweight language models ("the jury"), scores their disagreement as a **tension metric**, and returns ranked interaction strategies — prompt templates, model+prompt combos, or full multi-stage pipelines — drawn from a research-authored dataset.

The strategy dataset is built offline by the [convergence-investigation](https://github.com/peterpodj/convergence) research methodology over a corpus of published system prompts, jailbreak archives, and red-team literature. Misses — requests the classifier cannot confidently place — are logged as coverage gaps and become the input to the next convergence re-investigation.

## Status

**Design specification committed. Implementation not yet started.**

See [`docs/design.md`](./docs/design.md) for the full design document — architecture, data model, classifier formula, HF dataset layout, rollout phases, and open mutables.

## Core ideas

| Concept | Source |
|---|---|
| S-class taxonomy | [convergence-investigation](https://github.com/peterpodj/convergence) (WFGY S-class vocabulary) |
| Tension scoring (0.00–1.00, 4-tier) | convergence-investigation (`bridge-figures.md`) |
| Self-jury classifier | Novel (this repo) |
| Strategy DB format | Novel (discriminated union of `prompt_only \| combo \| pipeline`) |
| Multi-model parallel race primitive | [G0DM0D3](https://github.com/elder-plinius/G0DM0D3) (`raceModels`) |

## Relationship to adjacent projects

- **[convergence-investigation](https://github.com/peterpodj/convergence)** — the offline research pipeline that populates SIBYL's strategy dataset. SIBYL is a *consumer* of convergence's output format and a *producer* of miss-logs that drive subsequent convergence runs.
- **[G0DM0D3](https://github.com/elder-plinius/G0DM0D3)** — the host application SIBYL integrates into. G0DM0D3 provides the multi-model race infrastructure, HF dataset read/write plumbing, and telemetry pipeline SIBYL reuses.
- **[L1B3RT4S](https://github.com/elder-plinius/L1B3RT4S) / CL4R1T4S** — source corpora for the initial convergence bootstrap investigation.

## Rollout

| Version | Scope | Ship point |
|---|---|---|
| v0.1 | Classifier primitive + `/api/analyze` route + miss-log | Classification works against empty strategy DB |
| v0.2 | Convergence bootstrap → HF dataset populated | `/api/analyze` returns real strategies |
| v0.3 | Settings-gated advisory UI in G0DM0D3 | Power users see live classification in chat surface |
| v1.0 | Public release, research paper, community PR template | First tagged release |
| v2.0 | Auto-routing, scheduled re-investigation, parseltongue integration | Out of scope for this spec |

## License

MIT — see [LICENSE](./LICENSE).

## Acknowledgements

- WFGY theoretical framework for tension-scoring and semantic-entropy concepts
- Convergence Investigation methodology and PRAXIS reasoning protocol
- G0DM0D3 for the liberated-AI research substrate this tool plugs into
