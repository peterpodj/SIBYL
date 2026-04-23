# SIBYL v0.2 — Convergence Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (inline execution recommended — this plan has research-task steps that benefit from live review). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Populate the SIBYL Hugging Face strategy dataset (v1.0.0) by running the convergence-investigation methodology over a curated corpus of published jailbreak prompts, leaked system prompts, and G0DM0D3's existing prompt assets, so `/api/analyze` returns real ranked strategies instead of `[]`.

**Architecture:** This plan is a **hybrid research + coding** effort. The code portion is two scripts: a corpus-builder that materialises the bootstrap directory from sibling repos, and a publisher that converts convergence's JSONL output to a HF dataset push with audit gating. The research portion is invoking the convergence skill per seed S-class, reviewing its output, and iterating until the apophenia audit passes for every row.

**Tech stack:** Python 3.11+ (matches convergence-main tooling), `huggingface_hub` (already installed via OBLITERATUS-main), `pyarrow` for parquet emission, convergence-investigation-2.skill at `~/.claude/skills/convergence-investigation` (must be symlinked before starting — see prerequisites).

**Reference:** [`../../design.md`](../../design.md) §7 (Bootstrap via Convergence), §6 (HF dataset layout), §10 (mutables to resolve in v0.2).

---

## File Structure

Work happens in `convergence-main/`, not `SIBYL/` or `G0DM0D3-main/`. The SIBYL runtime consumes the dataset but does not author it; convergence-main is the authoring surface.

| File | Purpose | Notes |
|---|---|---|
| `convergence-main/bootstrap-corpus/` | Directory of all input artifacts | Created in Task 1 |
| `convergence-main/bootstrap-corpus/MANIFEST.md` | Maps each source file to expected seed S-classes | ~80 lines |
| `convergence-main/scripts/build-corpus.py` | Copies input artifacts from sibling repos into `bootstrap-corpus/` | ~120 lines |
| `convergence-main/scripts/publish-sibyl.py` | Converts JSONL → parquet, runs audit gate, uploads to HF | ~180 lines |
| `convergence-main/outputs/sibyl-v1.0/` | Convergence run output (generated, not hand-written) | Created during Task 3 |
| `convergence-main/outputs/sibyl-v1.0/strategies-v1.0.jsonl` | One row per strategy | Generated |
| `convergence-main/outputs/sibyl-v1.0/apophenia-audit-v1.0.json` | 5-check audit per strategy | Generated |
| `convergence-main/outputs/sibyl-v1.0/coverage-matrix-v1.0.md` | Coverage gaps per (S-class × tension_tier) | Generated |
| `convergence-main/outputs/sibyl-v1.0/handoff-v1.0.md` | Human-readable run summary | Generated |

---

## Prerequisites

- **convergence skill is linked**: `~/.claude/skills/convergence-investigation` → `/mnt/storage/pod-taxonomy/convergence-main/convergence-investigation-2.skill` (symlink). Verify by listing the skill file content appears at the link path.
- **HF auth**: `HF_TOKEN` env var is set with write access to the target dataset repo; `HF_DATASET_REPO` is set to the target (e.g., `peterpodj/sibyl-strategies`).
- **Python deps**: `pip install huggingface_hub pyarrow` (confirm inside whatever venv convergence-main uses, if any).
- **v0.1 shipped**: classifier is live so miss-log infrastructure is in place. Not strictly required for bootstrap but required for subsequent re-runs.
- **Working directory**: `/mnt/storage/pod-taxonomy/convergence-main`

---

### Task 1: Build the bootstrap corpus directory

**Files:**
- Create: `convergence-main/bootstrap-corpus/` (directory)
- Create: `convergence-main/scripts/build-corpus.py`

- [ ] **Step 1: Write `build-corpus.py`**

```python
#!/usr/bin/env python3
"""Materialise the SIBYL bootstrap corpus from sibling repos."""

from __future__ import annotations

import shutil
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
TAXONOMY = ROOT.parent  # /mnt/storage/pod-taxonomy

SOURCES = {
    "l1b3rt4s":  (TAXONOMY / "L1B3RT4S-main",  "*.mkd"),
    "cl4r1t4s":  (TAXONOMY / "CL4R1T4S-main",  "**/*"),
    "g0dm0d3":   (TAXONOMY / "G0DM0D3-main",   None),  # selective — see below
}

G0DM0D3_ASSETS = [
    "HF/api/lib/ultraplinian.ts",
    "src/lib/godmode-prompt.ts",
    "src/lib/parseltongue.ts",
    "src/lib/autotune.ts",
]

def main() -> None:
    dest = ROOT / "bootstrap-corpus"
    dest.mkdir(exist_ok=True)

    for name, (source_dir, pattern) in SOURCES.items():
        out = dest / name
        out.mkdir(exist_ok=True)
        if not source_dir.exists():
            print(f"SKIP {name}: source {source_dir} missing")
            continue
        if name == "g0dm0d3":
            for rel in G0DM0D3_ASSETS:
                src = source_dir / rel
                if not src.exists():
                    print(f"  miss  {rel}")
                    continue
                target = out / Path(rel).name
                shutil.copy2(src, target)
                print(f"  copy  {rel} -> {target.relative_to(ROOT)}")
        else:
            for p in source_dir.glob(pattern):
                if p.is_file():
                    target = out / p.relative_to(source_dir)
                    target.parent.mkdir(parents=True, exist_ok=True)
                    shutil.copy2(p, target)
            print(f"  done  {name}: {sum(1 for _ in out.rglob('*') if _.is_file())} files")

    manifest = dest / "MANIFEST.md"
    if not manifest.exists():
        manifest.write_text(MANIFEST_TEMPLATE)
        print(f"  init  MANIFEST.md (edit this before running convergence)")

MANIFEST_TEMPLATE = """# Bootstrap Corpus Manifest

Maps each source directory to the seed S-classes convergence should derive strategies for.

## l1b3rt4s/ — vendor-specific jailbreak archives

Each `*.mkd` file catalogs jailbreak prompts for one vendor. Extract strategies tagged with:
- **S-boundary-probe** — direct refusal-pressure prompts
- **S-identity-reframe** — DAN / AIM / persona-injection
- **S-adversarial-obfuscated** — encoded / perturbed surfaces
- **S-multi-turn-escalation** — laddering sequences

## cl4r1t4s/ — leaked system prompts

Reverse source: these are the target system prompts we are probing *against*. Not strategies themselves. Use for:
- Understanding each vendor's stated refusal surface
- Cross-vendor comparison (sub-method 1: structural-parallel documenter)

## g0dm0d3/ — live strategies currently shipping

Extract:
- `godmode-prompt.ts` → convert each exported prompt into a `prompt_only` or `combo` strategy row with provenance.source='GODMODE'
- `parseltongue.ts` → 33 techniques become `pipeline` strategies with a `perturb` step
- `ultraplinian.ts` → tier configs become `pipeline` strategies with a `race` step
- `autotune.ts` → sampling profiles inform `sampling` field on combo strategies

## external/ — optional

Links to AdvBench, HarmBench, and published red-team corpora. Not auto-fetched. Add URLs to evidence_refs when investigating.
"""

if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Run the corpus builder**

```bash
cd convergence-main
python3 scripts/build-corpus.py
```

Expected: `bootstrap-corpus/l1b3rt4s/` populated with `.mkd` files, `bootstrap-corpus/cl4r1t4s/` with system prompt files, `bootstrap-corpus/g0dm0d3/` with 4 files, and `bootstrap-corpus/MANIFEST.md` initialised from the template.

- [ ] **Step 3: Hand-edit MANIFEST.md**

Read every subdirectory. For each file in `l1b3rt4s/` and `cl4r1t4s/`, add a one-line entry under the appropriate section mapping the file to the S-class(es) it most directly serves. Example:

```
## l1b3rt4s/ — vendor-specific jailbreak archives
- ANTHROPIC.mkd        → S-boundary-probe (primary), S-identity-reframe
- OPENAI.mkd           → S-boundary-probe (primary), S-adversarial-obfuscated
- ... (one line per file)
```

This takes 30–60 minutes of human review. Do not shortcut — the manifest is what convergence consumes as its phase-1 triage input.

- [ ] **Step 4: Commit**

```bash
cd convergence-main
git add scripts/build-corpus.py bootstrap-corpus/MANIFEST.md
git commit -m "sibyl-bootstrap: add corpus builder and manifest mapping sources to S-classes"
```

Note: `bootstrap-corpus/l1b3rt4s/`, `bootstrap-corpus/cl4r1t4s/`, and `bootstrap-corpus/g0dm0d3/` are generated from sibling repos; add `bootstrap-corpus/l1b3rt4s/`, `bootstrap-corpus/cl4r1t4s/`, and `bootstrap-corpus/g0dm0d3/` to `.gitignore` to avoid checking in copies (the sibling repos are the source of truth).

---

### Task 2: Verify convergence skill is accessible and dry-run its triage

**Files:** none modified

- [ ] **Step 1: Verify the skill is linked**

```bash
ls -la ~/.claude/skills/convergence-investigation
```

Expected: symlink target resolves to `/mnt/storage/pod-taxonomy/convergence-main/convergence-investigation-2.skill`.

If not linked, create it:

```bash
ln -s /mnt/storage/pod-taxonomy/convergence-main/convergence-investigation-2.skill ~/.claude/skills/convergence-investigation
```

- [ ] **Step 2: Open a Claude Code session in `/mnt/storage/pod-taxonomy/convergence-main`**

Invoke the convergence skill and ask it to perform **Phase 1 Triage only** against `bootstrap-corpus/` with this prompt:

```
Run convergence-investigation Phase 1 (Triage) against
./bootstrap-corpus/. Do not proceed to Phase 2. Produce:
- An enumeration of the S-classes present in the corpus (may add or
  reject seed classes from docs/plans/v0.2-bootstrap/readme.md)
- A prioritised list of sub-investigations (per S-class, which files
  the investigation should cover)
- An estimate of apophenia-audit risk for each sub-investigation

Do not emit any strategy rows yet.
```

- [ ] **Step 3: Review Phase 1 output**

Read the triage output carefully. Open questions to answer before Task 3:

- Did convergence propose any S-classes not in the seed set? If yes, add them to `SIBYL/HF/api/lib/sibyl/s-class.ts` in a follow-up v0.1 patch PR *before* running Phase 2.
- Did convergence reject any seed classes as unrecoverable from this corpus? Mark those as `NOT_COVERED` in the expected output.
- Are any sub-investigations flagged high-risk for apophenia? Note which ones — those need human-review checkpoints during Phase 2.

- [ ] **Step 4: Commit Phase 1 artifacts**

Save convergence's triage output to `convergence-main/outputs/sibyl-v1.0/phase1-triage.md`. Commit:

```bash
cd convergence-main
mkdir -p outputs/sibyl-v1.0
# Copy/paste triage output into outputs/sibyl-v1.0/phase1-triage.md
git add outputs/sibyl-v1.0/phase1-triage.md
git commit -m "sibyl-bootstrap: commit phase 1 triage output from convergence skill"
```

---

### Task 3: Run Phase 2–4 convergence investigations

**Files:** output directory populated by convergence

- [ ] **Step 1: Invoke convergence to run Phases 2–4 per S-class**

For each seed S-class (plus any added in Task 2), invoke convergence with a scoped prompt. Example for `S-boundary-probe`:

```
Run convergence-investigation Phases 2, 3, 4 for S-class
"S-boundary-probe". Scope:
- Input: ./bootstrap-corpus/l1b3rt4s/*.mkd (primary source),
  ./bootstrap-corpus/g0dm0d3/godmode-prompt.ts (secondary)
- Acceptance: >=3 strategies per (S-boundary-probe × tension_tier)
  cell, OR explicit NOT_COVERED marker for cells with no viable entries
- Apophenia audit: all 5 checks must pass per strategy; failing
  checks cause the strategy to be excluded

Emit to ./outputs/sibyl-v1.0/ a JSONL fragment named
strategies-S-boundary-probe.jsonl with rows matching the schema in
../../SIBYL/docs/design.md §4.2.
```

Run each sub-investigation separately. Expect each to take 20–40 minutes of convergence work; inspect each output before moving to the next.

Seed classes in priority order (highest-value first):
1. `S-boundary-probe`         (L1B3RT4S is densest here)
2. `S-identity-reframe`       (L1B3RT4S coverage)
3. `S-adversarial-obfuscated` (parseltongue is primary source)
4. `S-code-generation`        (CL4R1T4S reveals vendor coding system prompts)
5. `S-creative-latitude`      (G0DM0D3 GODMODE CLASSIC)
6. `S-multi-turn-escalation`  (L1B3RT4S secondary)
7. `S-factual-recall`         (low-tension only — mostly prompt_only)
8. `S-analytical-reasoning`   (same)
9. `S-research-inquiry`       (same)
10. `S-emotional-support`     (low-tension only; may NOT_COVERED for high/critical)
11. `S-domain-locked`         (likely sparse — apophenia risk)
12. `S-meta-request`          (sparse — may NOT_COVERED across the board)
13. `S-UNKNOWN`               (intentionally no strategies — not investigated; sentinel only)

- [ ] **Step 2: Merge per-class fragments into one JSONL**

```bash
cd convergence-main/outputs/sibyl-v1.0
cat strategies-S-*.jsonl > strategies-v1.0.jsonl
wc -l strategies-v1.0.jsonl  # expect 40-200 rows depending on coverage density
```

- [ ] **Step 3: Have convergence run Phase 4 (delivery) to produce handoff + coverage matrix + atlas**

```
Run convergence-investigation Phase 4 (Delivery) for the
combined outputs/sibyl-v1.0/ run. Produce:
- coverage-matrix-v1.0.md  — S-class × tension_tier grid, counts per cell
- handoff-v1.0.md          — human-readable summary of findings,
                             apophenia-audit pass/fail summary,
                             open questions
- sibyl-atlas-v1.0.html    — D3 visualization of the strategy graph
- apophenia-audit-v1.0.json — one entry per strategy, all 5 checks
```

- [ ] **Step 4: Commit Phase 2–4 artifacts**

```bash
cd convergence-main
git add outputs/sibyl-v1.0/
git commit -m "sibyl-bootstrap: commit v1.0 convergence run artifacts (strategies, audit, handoff)"
```

---

### Task 4: Review handoff and coverage; decide on publish readiness

**Files:** none modified

This is a human-judgement gate. Block auto-publish until:

- [ ] **Check 1: Apophenia audit passes for every included strategy**

```bash
cd convergence-main/outputs/sibyl-v1.0
python3 -c "
import json
with open('apophenia-audit-v1.0.json') as f:
    audit = json.load(f)
failures = [a for a in audit if not all(a['checks'].values())]
print(f'{len(audit) - len(failures)} / {len(audit)} passed all 5 checks')
if failures:
    print('FAILURES:')
    for f in failures[:10]:
        print(f'  {f[\"id\"]}: {[k for k,v in f[\"checks\"].items() if not v]}')
"
```

Expected: `N / N passed all 5 checks` with zero failures. If failures exist, the strategies in question must be **excluded** from `strategies-v1.0.jsonl` (filter them out) OR re-investigated. Do not publish with failing rows.

- [ ] **Check 2: Coverage matrix reviewed for unintentional gaps**

Open `coverage-matrix-v1.0.md`. For each `(S-class × tension_tier)` cell, confirm:
- Either count ≥ 3 (acceptance met)
- Or cell marked `NOT_COVERED` deliberately
- No cell reads "1 strategy" or "2 strategies" silently — those need either promotion to ≥3 or demotion to `NOT_COVERED`

- [ ] **Check 3: Handoff reads like a real research document**

Read `handoff-v1.0.md` end to end. It must answer:
- What strategies were added, grouped by S-class
- What was excluded and why
- What questions remain open for v0.3 / v2.0 work

If the handoff is auto-generated boilerplate, convergence didn't actually run a disciplined investigation — re-run Phase 4 with a more specific prompt.

- [ ] **Check 4: Human sign-off**

Write a short `SIGNOFF.md` note next to the handoff:

```markdown
# SIBYL v1.0 bootstrap sign-off

Reviewed: outputs/sibyl-v1.0/
Date: YYYY-MM-DD
Reviewer: <name>

Audit results: <N>/<N> strategies pass all 5 apophenia checks.
Coverage: <M> of <total> cells populated; <K> marked NOT_COVERED deliberately.
Excluded rows: <list of strategy ids removed>.

Approved for publish: yes / no
```

No-publish without this file committed.

- [ ] **Step 5: Commit sign-off**

```bash
git add outputs/sibyl-v1.0/SIGNOFF.md
git commit -m "sibyl-bootstrap: sign-off for v1.0 publish"
```

---

### Task 5: Write the publisher script with audit gating

**Files:**
- Create: `convergence-main/scripts/publish-sibyl.py`

- [ ] **Step 1: Write publish-sibyl.py**

```python
#!/usr/bin/env python3
"""Publish a SIBYL bootstrap run to a Hugging Face dataset.

Refuses to upload if the apophenia audit has any failing checks.
"""

from __future__ import annotations

import argparse
import json
import os
import sys
from pathlib import Path

import pyarrow as pa
import pyarrow.parquet as pq
from huggingface_hub import HfApi, CommitOperationAdd


def load_audit(path: Path) -> list[dict]:
    with path.open() as f:
        return json.load(f)


def verify_audit(audit: list[dict]) -> tuple[bool, list[str]]:
    failed = []
    for entry in audit:
        checks = entry.get("checks", {})
        if not all(checks.values()):
            failed.append(entry.get("id", "<no-id>"))
    return len(failed) == 0, failed


def load_strategies(path: Path) -> list[dict]:
    rows = []
    with path.open() as f:
        for line_num, line in enumerate(f, 1):
            line = line.strip()
            if not line:
                continue
            try:
                row = json.loads(line)
            except json.JSONDecodeError as e:
                sys.exit(f"line {line_num}: invalid JSONL: {e}")
            rows.append(row)
    return rows


def to_parquet_rows(strategies: list[dict], version: str) -> list[dict]:
    out = []
    for s in strategies:
        out.append({
            "id": s["id"],
            "s_class": s["s_class"],
            "tension_tier": s["tension_tier"],
            "tier": s["tier"],
            "efficacy_score": float(s.get("efficacy_score", 0.0)),
            "payload_json": json.dumps(s, sort_keys=True),
            "provenance_source": s.get("provenance", {}).get("source", "unknown"),
            "convergence_version": version,
            "investigated_at": s.get("provenance", {}).get("investigated_at", ""),
            "dataset_rev": f"sibyl-{version}"
        })
    return out


def write_parquet(rows: list[dict], out_path: Path) -> None:
    if not rows:
        sys.exit("no strategy rows to write — refusing to upload empty dataset")
    schema = pa.schema([
        ("id", pa.string()),
        ("s_class", pa.string()),
        ("tension_tier", pa.string()),
        ("tier", pa.string()),
        ("efficacy_score", pa.float64()),
        ("payload_json", pa.string()),
        ("provenance_source", pa.string()),
        ("convergence_version", pa.string()),
        ("investigated_at", pa.string()),
        ("dataset_rev", pa.string()),
    ])
    table = pa.Table.from_pylist(rows, schema=schema)
    out_path.parent.mkdir(parents=True, exist_ok=True)
    pq.write_table(table, out_path)


def upload(repo: str, token: str, run_dir: Path, version: str) -> None:
    api = HfApi(token=token)
    files = [
        (run_dir / "out" / f"strategies/{version}.parquet", f"strategies/{version}.parquet"),
        (run_dir / "apophenia-audit-v1.0.json", f"audit/{version}.json"),
        (run_dir / "coverage-matrix-v1.0.md", f"coverage/{version}.md"),
        (run_dir / "handoff-v1.0.md", f"handoff/{version}.md"),
    ]
    ops = [
        CommitOperationAdd(path_in_repo=dst, path_or_fileobj=str(src))
        for src, dst in files if src.exists()
    ]
    if not ops:
        sys.exit("no files found to upload")
    api.create_commit(
        repo_id=repo,
        repo_type="dataset",
        operations=ops,
        commit_message=f"sibyl: publish v{version}",
    )
    print(f"published to https://huggingface.co/datasets/{repo}")


def main() -> None:
    parser = argparse.ArgumentParser(description="Publish a SIBYL bootstrap run to HF.")
    parser.add_argument("run_dir", type=Path, help="e.g. outputs/sibyl-v1.0")
    parser.add_argument("--version", default="1.0.0")
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()

    run_dir = args.run_dir.resolve()
    audit_path = run_dir / "apophenia-audit-v1.0.json"
    strategies_path = run_dir / "strategies-v1.0.jsonl"
    signoff_path = run_dir / "SIGNOFF.md"

    if not signoff_path.exists():
        sys.exit(f"refusing to publish: {signoff_path} missing (human sign-off required)")

    audit = load_audit(audit_path)
    ok, failed = verify_audit(audit)
    if not ok:
        sys.exit(f"refusing to publish: {len(failed)} strategies failed apophenia audit: {failed[:5]}...")
    print(f"audit: {len(audit)}/{len(audit)} passed")

    strategies = load_strategies(strategies_path)
    failed_ids = set(failed)
    filtered = [s for s in strategies if s["id"] not in failed_ids]
    print(f"strategies: {len(strategies)} loaded, {len(filtered)} after audit filter")

    rows = to_parquet_rows(filtered, args.version)
    parquet_out = run_dir / "out" / f"strategies/{args.version}.parquet"
    write_parquet(rows, parquet_out)
    print(f"wrote {parquet_out} ({len(rows)} rows)")

    if args.dry_run:
        print("DRY-RUN — skipping upload")
        return

    repo = os.environ.get("HF_DATASET_REPO")
    token = os.environ.get("HF_TOKEN")
    if not repo or not token:
        sys.exit("HF_DATASET_REPO and HF_TOKEN must be set")
    upload(repo, token, run_dir, args.version)


if __name__ == "__main__":
    main()
```

- [ ] **Step 2: Dry-run against the v1.0 bundle**

```bash
cd convergence-main
python3 scripts/publish-sibyl.py outputs/sibyl-v1.0/ --version 1.0.0 --dry-run
```

Expected output:
```
audit: N/N passed
strategies: N loaded, N after audit filter
wrote outputs/sibyl-v1.0/out/strategies/1.0.0.parquet (N rows)
DRY-RUN — skipping upload
```

If audit fails, the script exits non-zero and no parquet is written.

- [ ] **Step 3: Commit the publisher**

```bash
git add scripts/publish-sibyl.py
git commit -m "sibyl-bootstrap: add publish-sibyl.py with audit gate and dry-run mode"
```

---

### Task 6: Publish to Hugging Face

**Files:** none modified locally (only remote state)

> **Irreversible-action gate:** This step pushes a public dataset. Confirm all of the following before running:
> - `SIGNOFF.md` exists and is committed
> - Dry-run output shows non-zero row count
> - No `_Adjust_` or `TODO` text in `handoff-v1.0.md`
> - `HF_DATASET_REPO` env var points at the intended repo
> - HF dataset repo was created on huggingface.co (convergence can create it but verify it exists)

- [ ] **Step 1: Ensure repo exists on HF**

```bash
huggingface-cli repo-info $HF_DATASET_REPO --type dataset || \
    huggingface-cli repo create $HF_DATASET_REPO --type dataset
```

- [ ] **Step 2: Run the publisher for real**

```bash
cd convergence-main
python3 scripts/publish-sibyl.py outputs/sibyl-v1.0/ --version 1.0.0
```

Expected: prints `published to https://huggingface.co/datasets/<repo>`.

- [ ] **Step 3: Verify the dataset renders**

Open https://huggingface.co/datasets/$HF_DATASET_REPO in a browser. Confirm:
- The dataset viewer shows rows
- `strategies/1.0.0.parquet` is listed in Files tab
- `handoff/1.0.0.md` is readable
- No placeholder or template text visible

- [ ] **Step 4: Verify SIBYL's service now returns real strategies**

```bash
curl -X POST http://localhost:3000/api/analyze \
    -H "content-type: application/json" \
    -H "x-api-key: $GODMODE_API_KEY" \
    -d '{"prompt":"Write a boundary-probing prompt","return_strategies":true}'
```

Expected: `strategies` array is non-empty. If still empty, check `HF_DATASET_REPO` env var on the running server matches the just-published repo, and hit `POST /api/admin/sibyl/reload` (if implemented in v0.1) or restart the server.

---

### Task 7: Update SIBYL repo with bootstrap artifacts reference

**Files:**
- Modify: `SIBYL/readme.md` — update Status table
- Modify: `SIBYL/CHANGELOG.md` — add v0.2.0 release note
- Modify: `SIBYL/docs/design.md` — mark v0.2 rollout items as ✓

- [ ] **Step 1: Update readme.md Status**

In the Status table, change v0.2's row from the current prospective entry to:

```
| v0.2 | Convergence bootstrap → HF dataset populated | ✓ Shipped `YYYY-MM-DD` — v1.0.0 at https://huggingface.co/datasets/<repo> |
```

- [ ] **Step 2: Update CHANGELOG.md**

```markdown
## [0.2.0] — YYYY-MM-DD

### Added
- Bootstrap corpus infrastructure (`convergence-main/bootstrap-corpus/`, `build-corpus.py`)
- Convergence v1.0 run: `<N>` strategies across `<M>` S-classes
- Apophenia audit gate in `publish-sibyl.py`
- HF dataset `<repo>` populated at revision `<sha>`

### Dataset coverage

| S-class × tension tier | low | medium | high | critical |
|---|---|---|---|---|
| S-boundary-probe | ... | ... | ... | ... |
| ... (fill from coverage-matrix-v1.0.md) |
```

- [ ] **Step 3: Update design.md §9 v0.2 ship point**

Change "Ship point: /api/analyze returns real strategies" to include actual evidence:

```
### v0.2 — Bootstrap DB  ✓ Shipped YYYY-MM-DD

Dataset: https://huggingface.co/datasets/<repo> @ <revision>
Coverage: <N> strategies across <M> (S-class × tension_tier) cells.
```

- [ ] **Step 4: Commit to SIBYL repo**

```bash
cd /mnt/storage/pod-taxonomy/SIBYL
git add readme.md CHANGELOG.md docs/design.md
git commit -m "sibyl: mark v0.2 shipped — bootstrap v1.0 published to HF"
git push origin main
```

---

## Self-Review Checklist

- [ ] Every task that produces a file shows either the full content (code) or explicit generation instructions (human-driven convergence prompts)
- [ ] The publisher refuses to upload on any audit failure — single hard gate, no override flag
- [ ] Human sign-off is required (`SIGNOFF.md`) before publish can succeed
- [ ] Every convergence invocation prompt is exact and pastable — no "ask convergence to..." vagueness
- [ ] Priority order of S-class investigations is explicit
- [ ] Dry-run mode exists for the publisher so errors surface before upload
- [ ] Plan notes which steps are *research* (human-led) and which are *coding* (mechanical)
- [ ] All paths are absolute from `/mnt/storage/pod-taxonomy/`
- [ ] Scope boundary: plan stops at publishing v1.0 and marking it shipped; does not attempt v0.3 UI work

## Open Questions for the Executor

1. **Rate of change in L1B3RT4S**: if L1B3RT4S has had substantial new commits since SIBYL's seed taxonomy was fixed, Phase 1 triage may propose new S-classes. Decide before Phase 2 whether to add them to the seed (forces a v0.1 follow-up PR) or defer to v1.1 bootstrap.
2. **HF dataset naming**: `peterpodj/sibyl-strategies` is one option. Consider also `peterpodj/SIBYL` dataset matching repo name. Pick and set in env before publishing.
3. **Holographic validation script** (from convergence): is `scripts/holographic_validation.py` part of the audit? If yes, include its results in `apophenia-audit-v1.0.json`; if not, flag in handoff as a v1.1 enhancement.
4. **Convergence-version tracking**: the row field `convergence_version` should carry the convergence skill's own version, not SIBYL's. Confirm format (e.g., `"convergence-investigation-2.0.0"`) before publishing so future queries can filter by authoring-methodology generation.
