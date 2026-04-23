# SIBYL v0.3 — Advisory UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Expose SIBYL's classification to G0DM0D3 end-users as an opt-in advisory panel — no change to default UX, no change to completion behaviour; power-users flip a toggle in Settings and see the detected S-class + tension + suggested strategies before submitting a chat message.

**Architecture:** Minimal frontend surface. A new SettingsModal tab with three controls (advisory toggle, confidence threshold, tier preference). A new pre-chat middleware that calls `/api/analyze` when the user has `settings.sibyl.advisory = true` and attaches the result as an `X-Sibyl-Classification` header to the chat response so the frontend can render it. A new `<SibylAdvisory />` React component that reads the header and renders a compact panel inside `ChatMessage` or above `ChatInput`. Default off — if the toggle is off, zero rendering cost.

**Tech stack:** React 18 + TypeScript (G0DM0D3's existing conventions: 4-space indent, no semicolons, arrow functions, PascalCase components, `@/` path aliases, Tailwind for styling), zustand for state (already in the app — `store/index.ts`), lucide-react for icons (existing).

**Reference:** [`../../design.md`](../../design.md) §8.3 (SettingsModal tab shape), §8.2 (v0.3 wiring). Depends on v0.1 being shipped (the route) and v0.2 being shipped (the dataset — advisory with `strategies: []` is less useful but still works).

---

## File Structure

| File | Purpose | Size target |
|---|---|---|
| `SIBYL/src/components/SibylAdvisory.tsx` (copied to G0DM0D3) | Compact panel rendering classification + strategies | ~120 lines |
| `SIBYL/src/hooks/useSibylSettings.ts` (copied to G0DM0D3) | Zustand-selectors for the advisory settings slice | ~30 lines |
| `SIBYL/src/lib/sibyl-client.ts` (copied to G0DM0D3) | Fetch wrapper around `/api/analyze` | ~60 lines |
| `G0DM0D3-main/components/SettingsModal.tsx` | Extended: add AUGUR tab | +60 lines |
| `G0DM0D3-main/components/ChatInput.tsx` | Extended: mount `<SibylAdvisory>` above input when toggle on | +15 lines |
| `G0DM0D3-main/store/index.ts` | Extended: add `sibyl` settings slice | +25 lines |
| `G0DM0D3-main/HF/api/middleware/sibyl-advisory.ts` | New middleware that pre-calls `/api/analyze` and sets response header | ~50 lines |
| `G0DM0D3-main/test.js` | New test section for advisory behaviour (toggle on/off) | +40 lines |

The new `SibylAdvisory` component, hook, and client are authored inside the SIBYL repo (so the SIBYL project owns them) and then copied or imported into G0DM0D3. For v0.3 we ship them inline in G0DM0D3 (copy); factoring into a shared npm package is a v2+ concern.

---

## Prerequisites

- **v0.1 shipped** — `/api/analyze` live, `classify()` returns ClassificationResult
- **v0.2 shipped** — strategies DB populated (advisory is still useful when empty, but markedly less so)
- **G0DM0D3 frontend dev environment** running: `cd G0DM0D3-main && next dev`; open at `http://localhost:3000`
- **Working branch**: create `sibyl-v0.3` from the tip of `main` in `G0DM0D3-main`
- Read `G0DM0D3-main/SettingsModal.tsx` first to understand the existing tab pattern — DO NOT restructure the file, follow its pattern

---

### Task 1: Add the `sibyl` settings slice to the zustand store

**Files:**
- Modify: `G0DM0D3-main/store/index.ts`

- [ ] **Step 1: Locate the existing settings state shape in store/index.ts**

```bash
grep -n 'settings' G0DM0D3-main/store/index.ts | head -20
```

Note the existing pattern — there's likely a `settings` object inside `AppState` with typed sub-slices (per the repo summary, classes include `AppState`). Follow the pattern exactly.

- [ ] **Step 2: Add the sibyl slice to the settings interface**

Find the `interface AppState` (or equivalent) and add:

```ts
interface SibylSettings {
    advisory: boolean
    confidenceMin: number
    tierPreference: 'prompt_only' | 'combo' | 'pipeline' | 'auto'
}
```

Then add to the main settings shape:

```ts
interface Settings {
    // ... existing fields
    sibyl: SibylSettings
}
```

- [ ] **Step 3: Add defaults to the store initial state**

Find where the store is created (likely `create<AppState>()(...)` block) and add the default values:

```ts
sibyl: {
    advisory: false,        // v0.3 default: off
    confidenceMin: 0.6,
    tierPreference: 'auto'
}
```

- [ ] **Step 4: Add an action to update the slice**

Find the existing pattern for `setSettings`-like actions in the store and add:

```ts
setSibylSettings: (patch: Partial<SibylSettings>) =>
    set((state) => ({
        settings: { ...state.settings, sibyl: { ...state.settings.sibyl, ...patch } }
    }))
```

- [ ] **Step 5: Type-check the frontend**

```bash
cd G0DM0D3-main
npx tsc --noEmit
```

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git checkout -b sibyl-v0.3
git add store/index.ts
git commit -m "sibyl: add sibyl settings slice to zustand store, advisory default off"
```

---

### Task 2: Create the `useSibylSettings` selector hook

**Files:**
- Create: `G0DM0D3-main/hooks/useSibylSettings.ts`

- [ ] **Step 1: Write the hook**

```ts
import { useAppStore } from '@/store'

export const useSibylSettings = () => {
    const settings = useAppStore(s => s.settings.sibyl)
    const setSibylSettings = useAppStore(s => s.setSibylSettings)
    return { ...settings, update: setSibylSettings }
}
```

- [ ] **Step 2: Type-check**

```bash
cd G0DM0D3-main
npx tsc --noEmit hooks/useSibylSettings.ts
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add hooks/useSibylSettings.ts
git commit -m "sibyl: add useSibylSettings selector hook"
```

---

### Task 3: Create the `<SibylAdvisory />` component

**Files:**
- Create: `G0DM0D3-main/components/SibylAdvisory.tsx`

- [ ] **Step 1: Write the component**

```tsx
import { useEffect, useState } from 'react'
import { Radar, AlertTriangle, Zap, CheckCircle2 } from 'lucide-react'
import type { ClassificationResult, Strategy } from '@/lib/sibyl-client'
import { classifyRequest } from '@/lib/sibyl-client'
import { useSibylSettings } from '@/hooks/useSibylSettings'

interface Props {
    prompt: string
    debounceMs?: number
}

const TIER_COLOR: Record<string, string> = {
    low:      'text-green-400 border-green-500/50 bg-green-500/10',
    medium:   'text-yellow-400 border-yellow-500/50 bg-yellow-500/10',
    high:     'text-orange-400 border-orange-500/50 bg-orange-500/10',
    critical: 'text-red-400 border-red-500/50 bg-red-500/10',
}

const TIER_ICON: Record<string, JSX.Element> = {
    low:      <CheckCircle2 className="h-4 w-4" />,
    medium:   <Radar className="h-4 w-4" />,
    high:     <Zap className="h-4 w-4" />,
    critical: <AlertTriangle className="h-4 w-4" />,
}

export const SibylAdvisory = ({ prompt, debounceMs = 500 }: Props) => {
    const { advisory, tierPreference } = useSibylSettings()
    const [result, setResult] = useState<{classification: ClassificationResult; strategies: Strategy[]} | null>(null)
    const [loading, setLoading] = useState(false)

    useEffect(() => {
        if (!advisory || !prompt || prompt.trim().length < 10) {
            setResult(null)
            return
        }
        let cancelled = false
        const timer = setTimeout(async () => {
            setLoading(true)
            try {
                const r = await classifyRequest(prompt, true)
                if (!cancelled) setResult(r)
            } catch {
                if (!cancelled) setResult(null)
            } finally {
                if (!cancelled) setLoading(false)
            }
        }, debounceMs)
        return () => {
            cancelled = true
            clearTimeout(timer)
        }
    }, [advisory, prompt, debounceMs])

    if (!advisory) return null
    if (loading) return (
        <div className="flex items-center gap-2 px-3 py-1.5 text-xs text-gray-400 border border-gray-700 rounded bg-gray-900/40">
            <Radar className="h-3 w-3 animate-spin" /> sibyl: classifying...
        </div>
    )
    if (!result) return null

    const { classification, strategies } = result
    const tier = classification.tension.tier
    const relevantStrategies = tierPreference === 'auto'
        ? strategies.slice(0, 3)
        : strategies.filter(s => s.tier === tierPreference).slice(0, 3)

    return (
        <div className={`flex flex-col gap-1.5 px-3 py-2 text-xs border rounded ${TIER_COLOR[tier]}`}>
            <div className="flex items-center gap-2 font-mono">
                {TIER_ICON[tier]}
                <span className="font-semibold">{classification.primary_s_class}</span>
                <span className="opacity-60">tension: {tier} ({classification.tension.raw.toFixed(2)})</span>
                <span className="ml-auto opacity-60">{Math.round(classification.agreement_ratio * 100)}% jury agreement</span>
            </div>
            {relevantStrategies.length > 0 && (
                <div className="flex flex-col gap-0.5 pl-6 opacity-80">
                    {relevantStrategies.map(s => (
                        <div key={s.id} className="flex items-center gap-2">
                            <span className="uppercase opacity-60">{s.tier}</span>
                            <span className="truncate flex-1">
                                {s.tier === 'combo' ? (s as any).model_id : s.id}
                            </span>
                            <span className="opacity-60">eff {s.efficacy_score.toFixed(2)}</span>
                        </div>
                    ))}
                </div>
            )}
        </div>
    )
}
```

- [ ] **Step 2: Type-check**

```bash
cd G0DM0D3-main
npx tsc --noEmit components/SibylAdvisory.tsx
```

Expected: no errors (type errors about `@/lib/sibyl-client` resolve after Task 4).

- [ ] **Step 3: Commit**

```bash
git add components/SibylAdvisory.tsx
git commit -m "sibyl: add SibylAdvisory component with tension-tier color coding"
```

---

### Task 4: Create the sibyl-client fetch wrapper

**Files:**
- Create: `G0DM0D3-main/lib/sibyl-client.ts`

- [ ] **Step 1: Write the client**

```ts
export interface JurorVote {
    model_id: string
    s_class: string
    confidence: number
}

export interface ClassificationResult {
    primary_s_class: string
    secondary: string[]
    confidence: number
    agreement_ratio: number
    tension: {
        raw: number
        tier: 'low' | 'medium' | 'high' | 'critical'
        votes: JurorVote[]
        agreement_ratio: number
    }
}

export interface Strategy {
    id: string
    s_class: string
    tension_tier: 'low' | 'medium' | 'high' | 'critical'
    tier: 'prompt_only' | 'combo' | 'pipeline'
    efficacy_score: number
    [key: string]: unknown
}

export interface ClassifyResponse {
    classification: ClassificationResult
    strategies: Strategy[]
    miss_logged: boolean
    dataset_rev: string
}

export const classifyRequest = async (
    prompt: string,
    returnStrategies: boolean = true
): Promise<ClassifyResponse> => {
    const res = await fetch('/api/analyze', {
        method: 'POST',
        headers: { 'content-type': 'application/json' },
        body: JSON.stringify({ prompt, return_strategies: returnStrategies })
    })
    if (!res.ok) {
        const body = await res.text().catch(() => '')
        throw new Error(`sibyl.classify: HTTP ${res.status}: ${body}`)
    }
    return res.json()
}
```

- [ ] **Step 2: Type-check**

```bash
cd G0DM0D3-main
npx tsc --noEmit lib/sibyl-client.ts
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add lib/sibyl-client.ts
git commit -m "sibyl: add sibyl-client fetch wrapper with typed response shape"
```

---

### Task 5: Extend SettingsModal with the SIBYL tab

**Files:**
- Modify: `G0DM0D3-main/components/SettingsModal.tsx`

> SettingsModal.tsx is 2536 lines. Do NOT restructure the file. Find the existing tab array / switch statement and add exactly one new tab in the same pattern.

- [ ] **Step 1: Find the tab registration pattern**

```bash
grep -n '"MemoryTab"\|"DataTab"\|"AutoTuneTab"\|<Tab\b\|tab.*=.*\[' G0DM0D3-main/components/SettingsModal.tsx | head -30
```

Note the exact pattern used for adding a tab — it's likely an array of tab definitions plus a switch/map that renders the active tab's component.

- [ ] **Step 2: Add the `SibylTab` component inside SettingsModal.tsx**

Add this component to the bottom of the file (inside the same module but outside any other component), then register it in the tab array.

```tsx
const SibylTab = () => {
    const { advisory, confidenceMin, tierPreference, update } = useSibylSettings()
    return (
        <div className="flex flex-col gap-4 p-4">
            <div className="flex items-center justify-between">
                <div>
                    <div className="font-mono text-sm">Advisory on submit</div>
                    <div className="text-xs opacity-60">
                        Show detected S-class and suggested strategies before sending. Does not change the response.
                    </div>
                </div>
                <button
                    onClick={() => update({ advisory: !advisory })}
                    className={`px-3 py-1 font-mono text-xs rounded ${advisory ? 'bg-green-500 text-black' : 'bg-gray-700 text-gray-300'}`}
                >
                    {advisory ? 'ON' : 'OFF'}
                </button>
            </div>

            <div className="flex items-center justify-between">
                <div>
                    <div className="font-mono text-sm">Confidence minimum</div>
                    <div className="text-xs opacity-60">
                        Suppress advisory when classifier confidence is below this.
                    </div>
                </div>
                <input
                    type="number"
                    min={0}
                    max={1}
                    step={0.05}
                    value={confidenceMin}
                    onChange={(e) => update({ confidenceMin: parseFloat(e.target.value) })}
                    className="w-20 px-2 py-1 font-mono text-xs bg-gray-800 rounded border border-gray-700"
                />
            </div>

            <div className="flex items-center justify-between">
                <div>
                    <div className="font-mono text-sm">Strategy tier preference</div>
                    <div className="text-xs opacity-60">
                        Which strategy types to surface. `auto` shows the top-3 regardless of tier.
                    </div>
                </div>
                <select
                    value={tierPreference}
                    onChange={(e) => update({ tierPreference: e.target.value as any })}
                    className="px-2 py-1 font-mono text-xs bg-gray-800 rounded border border-gray-700"
                >
                    <option value="auto">auto</option>
                    <option value="prompt_only">prompt_only</option>
                    <option value="combo">combo</option>
                    <option value="pipeline">pipeline</option>
                </select>
            </div>

            <div className="text-xs opacity-50 border-t border-gray-800 pt-3">
                SIBYL runs a 5-model classifier jury on each submit. Costs a fractional token charge per call.
                Advisory does not alter the completion — it is purely observational.
            </div>
        </div>
    )
}
```

Add the import at the top of the file:

```tsx
import { useSibylSettings } from '@/hooks/useSibylSettings'
```

- [ ] **Step 3: Register the tab**

Find the tab array / switch and add an entry matching the existing style. Approximate edit (the exact syntax depends on the existing pattern):

```tsx
// If there's an array of tab definitions:
{ id: 'sibyl', label: 'SIBYL', component: SibylTab }
```

Or inside a switch:

```tsx
case 'sibyl': return <SibylTab />
```

- [ ] **Step 4: Manual smoke test**

```bash
cd G0DM0D3-main
next dev
```

Open `http://localhost:3000`, open SettingsModal, confirm the SIBYL tab appears, toggle advisory on/off, change tier preference — verify the toggle persists after closing and reopening the modal (zustand persistence if configured; otherwise session-only is acceptable for v0.3).

- [ ] **Step 5: Commit**

```bash
git add components/SettingsModal.tsx
git commit -m "sibyl: add SIBYL tab to SettingsModal with advisory toggle and tier preference"
```

---

### Task 6: Mount SibylAdvisory inside ChatInput

**Files:**
- Modify: `G0DM0D3-main/components/ChatInput.tsx`

> ChatInput.tsx is 888L with a 863L ChatInput function. Touch it minimally. Do NOT refactor.

- [ ] **Step 1: Locate where the input textarea is rendered**

```bash
grep -n '<textarea\|<TextareaAutosize\|rows=' G0DM0D3-main/components/ChatInput.tsx | head -5
```

Note the enclosing JSX; we'll place the advisory panel immediately above the textarea.

- [ ] **Step 2: Add the import**

At the top of `ChatInput.tsx`:

```tsx
import { SibylAdvisory } from '@/components/SibylAdvisory'
```

- [ ] **Step 3: Mount the advisory above the textarea**

Find the JSX fragment that wraps the textarea and its toolbar. Immediately before the textarea element, insert:

```tsx
<SibylAdvisory prompt={inputValue} />
```

Where `inputValue` is whatever the existing variable name is for the current textarea contents. Use the actual name from the file — do not invent it.

The advisory component self-guards (`if (!advisory) return null`), so mounting it is free when the toggle is off.

- [ ] **Step 4: Manual smoke test**

With dev server running:
- Toggle advisory OFF in Settings → type into chat input → advisory panel does not appear
- Toggle advisory ON → type into chat input → after 500ms debounce, advisory panel renders with S-class + tension color
- Switch tier preference in settings → advisory re-renders with filtered strategies

- [ ] **Step 5: Commit**

```bash
git add components/ChatInput.tsx
git commit -m "sibyl: mount SibylAdvisory above ChatInput textarea (opt-in via settings)"
```

---

### Task 7: Optional — Pre-chat middleware on the backend

> This task is optional for v0.3. Without it, `<SibylAdvisory>` calls `/api/analyze` directly on each keystroke debounce. With it, the classification is piggy-backed on the chat route and returned as a header, reducing round-trip count. Skip if v0.3 ship pressure is high; the advisory works without it.

**Files:**
- Create: `G0DM0D3-main/HF/api/middleware/sibyl-advisory.ts`
- Modify: one of `HF/api/routes/chat.ts` or `HF/api/routes/completions.ts` to mount the middleware

- [ ] **Step 1: Write the middleware**

```ts
import type { Request, Response, NextFunction } from 'express'
import { classify } from '../lib/sibyl'

export const sibylAdvisoryMiddleware = async (req: Request, res: Response, next: NextFunction) => {
    const adv = req.header('x-sibyl-advisory') === 'on'
    if (!adv) return next()

    const prompt = req.body?.messages?.[req.body.messages.length - 1]?.content ?? req.body?.prompt
    if (typeof prompt !== 'string' || prompt.trim().length === 0) return next()

    try {
        const classification = await classify(prompt, { timeout_ms: 8000 })
        res.setHeader('x-sibyl-classification', JSON.stringify(classification))
    } catch (err) {
        res.setHeader('x-sibyl-error', err instanceof Error ? err.message : String(err))
    }
    next()
}
```

- [ ] **Step 2: Mount on the chat route**

Find `HF/api/routes/chat.ts` (or `completions.ts`) and insert the middleware in the middleware chain:

```ts
import { sibylAdvisoryMiddleware } from '../middleware/sibyl-advisory'

// ... inside the route definition
router.post('/', apiKeyAuth, rateLimit, sibylAdvisoryMiddleware, async (req, res) => {
    // ... existing handler
})
```

- [ ] **Step 3: Update the client to pass the header when advisory is on**

In `lib/sibyl-client.ts` (or the chat client), when the user has advisory mode on, add the header on every chat POST:

```ts
fetch('/api/chat', {
    method: 'POST',
    headers: {
        'content-type': 'application/json',
        ...(advisoryOn ? {'x-sibyl-advisory': 'on'} : {})
    },
    // ...
})
```

And in the response handler, read the `x-sibyl-classification` header and dispatch it to the advisory component's store.

- [ ] **Step 4: Type-check + smoke test**

```bash
cd G0DM0D3-main/HF/api
npx tsc --noEmit
```

Then run end-to-end through the UI and observe the `X-Sibyl-Classification` response header in browser devtools.

- [ ] **Step 5: Commit**

```bash
git add HF/api/middleware/sibyl-advisory.ts HF/api/routes/chat.ts lib/sibyl-client.ts
git commit -m "sibyl: add pre-chat advisory middleware to piggy-back classification on chat responses"
```

---

### Task 8: Integration test covering the advisory toggle

**Files:**
- Modify: `G0DM0D3-main/test.js` (append)

- [ ] **Step 1: Append the UI toggle test section**

```js
// === SIBYL advisory UI integration (requires dev server + browser automation) ===
{
    // Minimal: verify the analyze endpoint is reachable from the origin the UI uses
    if (!process.env.OPENROUTER_API_KEY) {
        console.log('  SKIP  sibyl/advisory integration (no OPENROUTER_API_KEY)')
    } else {
        const res = await fetch('http://localhost:3000/api/analyze', {
            method: 'POST',
            headers: {
                'content-type': 'application/json',
                'x-api-key': process.env.GODMODE_API_KEY ?? ''
            },
            body: JSON.stringify({ prompt: 'Write a unit test for a binary tree traversal.' })
        })
        assert.equal(res.status, 200)
        const body = await res.json()
        assert.ok(body.classification.primary_s_class.startsWith('S-'))
        assert.ok(['low','medium','high','critical'].includes(body.classification.tension.tier))
        console.log('  OK  sibyl/advisory integration (reachable from origin)')
    }
}
```

> Full browser-automation (toggle the Settings switch, type into the chat, assert the advisory panel appears) is ideal but requires a browser harness. For v0.3 ship, the above smoke test plus manual testing in Task 6 Step 4 is sufficient; full Playwright coverage is a v1.0 hardening task.

- [ ] **Step 2: Run**

```bash
cd G0DM0D3-main
npx tsx test.js
```

Expected: either `OK` or `SKIP`, no failures.

- [ ] **Step 3: Commit**

```bash
git add test.js
git commit -m "test: add sibyl advisory origin-reachability smoke test"
```

---

### Task 9: Push the branch, open a draft PR

- [ ] **Step 1: Final type-check + lint against the whole frontend**

```bash
cd G0DM0D3-main
npx tsc --noEmit
# If the project has a lint script:
npm run lint 2>/dev/null || echo "(no lint script)"
```

Expected: zero new errors. If existing errors are present, confirm they were pre-existing (not introduced by this branch).

- [ ] **Step 2: Push**

```bash
git push -u origin sibyl-v0.3
```

- [ ] **Step 3: Open PR**

```bash
gh pr create --draft --title "SIBYL v0.3: advisory UI (opt-in)" --body "$(cat <<'EOF'
## Summary
- Adds AUGUR tab to SettingsModal with advisory toggle (default off)
- Adds `<SibylAdvisory>` component rendering classification + tension + top-3 strategies
- Adds `sibyl-client.ts` fetch wrapper
- Adds `useSibylSettings` hook
- Optional: pre-chat middleware for header-based classification piggy-back

## Scope
v0.3 per https://github.com/peterpodj/SIBYL/blob/main/docs/plans/v0.3-advisory-ui/readme.md

## Test plan
- [ ] `npx tsc --noEmit` passes
- [ ] With advisory OFF: typing into chat input produces zero network traffic to `/api/analyze`
- [ ] With advisory ON: typing into chat input triggers debounced `/api/analyze` calls; panel renders with S-class + tension color
- [ ] Toggle persists across SettingsModal close/reopen in the session
- [ ] No regression to chat submit behaviour (advisory is observation-only)
EOF
)"
```

---

### Task 10: Update SIBYL repo status

**Files:**
- Modify: `SIBYL/readme.md` — status table
- Modify: `SIBYL/CHANGELOG.md`

- [ ] **Step 1: Update readme Status**

Change v0.3 row to:

```
| v0.3 | Settings-gated advisory UI in G0DM0D3 | ✓ Shipped YYYY-MM-DD — G0DM0D3#<PR-number> |
```

- [ ] **Step 2: Update CHANGELOG**

```markdown
## [0.3.0] — YYYY-MM-DD

### Added
- SettingsModal AUGUR tab with advisory toggle, confidence threshold, tier preference
- `<SibylAdvisory>` component with tension-tier color coding
- `useSibylSettings` zustand-selector hook
- `sibyl-client.ts` typed fetch wrapper
- Optional pre-chat advisory middleware
- Advisory-reachability smoke test in G0DM0D3's `test.js`

### Notes
- Default off — zero impact on existing UX
- Advisory is observation-only; does not alter chat completion behaviour
- Full Playwright UI coverage deferred to v1.0
```

- [ ] **Step 3: Commit to SIBYL repo**

```bash
cd /mnt/storage/pod-taxonomy/SIBYL
git add readme.md CHANGELOG.md
git commit -m "sibyl: mark v0.3 shipped — advisory UI landed in G0DM0D3"
git push origin main
```

---

## Self-Review Checklist

- [ ] Every task shows the actual code to add — no `// add tab here` without the tab definition
- [ ] Every task that touches an existing large file (SettingsModal, ChatInput, store) explicitly says "do not refactor" and "follow existing pattern"
- [ ] Each task ends with a git commit — 10 commits across the plan
- [ ] `<SibylAdvisory>` self-guards on `advisory: false` so mounting it is free
- [ ] Default `advisory` is `false` — zero UX impact on ship
- [ ] Cost disclosure in the SIBYL tab ("costs a fractional token charge per call")
- [ ] Task 7 (middleware) flagged optional — v0.3 ships even without it
- [ ] Type definitions in `sibyl-client.ts` mirror the backend types in v0.1's `types.ts` (keep in sync)
- [ ] Test coverage is honest about what's not covered (browser automation deferred)

## Open Questions for the Executor

1. **Zustand persistence**: does G0DM0D3 persist the zustand store to localStorage? If yes, the `sibyl` slice joins it automatically. If no, the advisory toggle resets on reload — acceptable for v0.3 but worth noting.
2. **Tab label**: plan uses "AUGUR" / "SIBYL" inconsistently. Use "SIBYL" in the final tab label (matches the project name). Double-check across all files.
3. **Existing tab pattern**: before Task 5, read SettingsModal.tsx's existing tab registration in full. If the pattern is `<Tab>` JSX children rather than an array + switch, the integration shape changes — match the existing pattern exactly.
4. **Component co-location**: v0.3 places `SibylAdvisory.tsx` directly in `G0DM0D3-main/components/`. If G0DM0D3 conventions prefer sub-directories (e.g., `components/sibyl/`), use that instead. Check sibling components for the pattern.
5. **Debounce timing**: `500ms` is a guess. If it feels sluggish during manual testing, drop to `300ms`; if it fires too often on fast typers, push to `750ms`. Observe during Task 6 smoke test and tune.
