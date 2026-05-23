# BidTrace — Project Memory

## What this is

BidTrace is an AI-assisted construction tender estimation tool for **Mirkwood Dynamics Restoration** (Ottawa, ON). It ingests tender documents (PDFs, specs, drawings, addenda, optional field video/audio), runs them through Claude to extract bid items, verifies the extraction with a multi-pass loop, matches items to the MDG price book, and produces an editable estimate ready for export to Excel.

Industry: concrete repair, parking garage rehabilitation, building envelope restoration. Most tenders come from consulting engineering firms (RJC, Exp Services, Morrison Hershfield, Concentric, etc.) for property managers in the Ottawa region.

## File structure

The project is intentionally a two-file deployment:

| File | Role |
|---|---|
| `BidTrace.html` | Single-page front-end app — all UI, app logic, AI calls, embedded MDG price book data. ~6,500 lines. |
| `bidtrace_all_in_one_worker.js` | Cloudflare Worker deployed at `bidtrace-proxy.dfried.workers.dev`. Proxies Anthropic API calls (so the API key never reaches the browser) and handles AssemblyAI transcription. Also embeds a serve-from-edge copy of `BidTrace.html` at its `GET /` route. ~6,400 lines. |

**The worker file contains a duplicate copy of the HTML.** Any logic change that affects extraction, verification, or row classification must be applied to **both** files. UI-only changes (CSS, layout HTML) only need to be in `BidTrace.html`, but if you modify the embedded copy too, keep them in sync.

## Architecture

### Front-end navigation (6 pages)
- **Dashboard** — KPIs, project list
- **New Estimate** — 5-step wizard: Project Info → Upload → Extract & Verify → QA Review → Estimate
- **Estimate** — editable line-item table with Excel export
- **Price Book Manager** — search/edit/import/export the MDG price book; web-search fallback for missing prices; pending shared-contributions feature
- **Regression** — accuracy test suite
- **Billing** — Free / Pro / Enterprise plan selection

### AI modules (inside `BidTrace.html`)

| Module | Purpose | Model |
|---|---|---|
| `BidTraceAI` | Scope report, quantity schedule extraction, OCR fallback, spec notes extraction | Sonnet 4.6, Haiku 4.5 |
| `BidTraceVerifier` | Multi-pass verification loop, chunked spec processing, accuracy scoring (target ≥ 97%) | Opus 4.7 (strict), Sonnet (balanced), Haiku (fast) — chosen by `chooseVerificationMode()` |
| `BidTraceMatching` | Map extracted rows to price book codes; benchmark-aligned semantic normalisation | Haiku 4.5 |
| `BidTraceExpertReview` | Final estimating review + spec compliance review | Opus 4.7 |

### State

Global `state` object (defined around line 3460 of `BidTrace.html`):

```js
state = {
  user, currentStep, projectInfo,
  uploadedFiles, takeoffSegments, takeoffPhotos,
  uploadedVideos, videoTranscriptSegments, videoFrames, combinedTakeoffSegments,
  extractedItems, projectFingerprint,
  scopeReport, specNotes,
  extractionIsReal, rowTypeAudit,
  instructionRows, productHintRows, estimateEligibleRows,
  qaItems, estimateItems,
  projects: [...] // dashboard tile data
}
```

Other globals:
- `PRICE_BOOK` — code→{labour, material, total} object used by the estimator
- `MDG` — full price book array for the Price Book Manager UI
- `MANDATORY_ESTIMATE_ITEMS` — overhead items that auto-prepend to every estimate

### Worker routes
- `GET /ping` — health check (reports whether Anthropic + AssemblyAI keys are set)
- `GET /` — serves the embedded HTML
- `POST /transcribe` — AssemblyAI audio transcription (used for video uploads and field audio)
- `POST /` (any other path) — proxies to `https://api.anthropic.com/v1/messages`, streams body through to avoid Cloudflare CPU timeouts on large Opus responses

Secrets live in Cloudflare Worker environment: `ANTHROPIC_API_KEY`, `ASSEMBLYAI_API_KEY`.

## Row classification — critical domain logic

Extracted rows are classified by `normalizeRowType()`. This determines whether a row appears on the **estimate table** or in the **spec notes / contractual items sidebar**.

**Estimate-eligible types** (appear in the QA table and estimate):
- `scope_item_row` — physical work with quantity + unit + price code
- `lump_sum_item_row` — LS items with a valid price code
- `unit_price_item_row` — explicit unit-price tender lines
- `allowance_row` — cash allowances on the bid form

**Non-estimate types** (sidebar only):
- `instruction_row` — bid form notes, contract terms, headers, subtotals
- `product_hint_row` — product references without quantities

### The verifier "contractual obligations" rule

The Opus verifier prompt explicitly forbids adding the following as `missed_items`, because they are **never standalone bid lines** — they are contract conditions priced into the unit rates:

- Bonds (bid, performance, labour & material payment)
- Insurance (CGL, professional liability, automobile, equipment, installation floater)
- OHSMS / WSIB / safety certifications
- Permits, utility clearances, inspections
- Warranty obligations
- Record drawings, as-built drawings, shop drawings
- Engineering certification, field reviews, consultant sign-off
- Site security, hoarding, temporary facilities (unless explicitly a bid line)
- Mobilization, demobilization, cleanup (unless explicitly a bid line)
- Taxes, overhead, profit, contingency markups

These belong in `method_notes_missed` (which surfaces in the spec instructions sidebar). If you change the verifier prompt, preserve this rule.

### Safety net for legacy verifier output

Even with the prompt rule, `normalizeRowType` has a final filter: any row where `source_type === 'verification_agent'` AND no valid price code (`!code || code === '' || code === 'NONE'`) is forced to `instruction_row`. This catches anything that slips through the prompt.

## QA Review layout (Step 4)

Two-column layout:
- **Left sidebar** (`#qa-sidebar`, 288px, collapsible to 38px) — three independently expandable sections:
  - 📄 Spec Instructions (`#spec-notes-panel`)
  - 📋 Scope Report (`#scope-report-panel`)
  - ⚖️ Contractual Items (`#contractual-panel`) — verifier-found, not priced
- **Right main** (`.qa-main`) — summary cards, gate message, full-width QA table, action buttons

Toggle functions: `toggleQASidebar()`, `toggleSidebarSection(bodyId, chevronId)`, `toggleScopeReport()`.

## Conventions for editing this codebase

1. **Sync HTML and worker.** Any change to extraction prompts, verifier prompts, row classification logic, or AI module code must be applied to both `BidTrace.html` and the embedded copy in `bidtrace_all_in_one_worker.js`.

2. **CSS variables.** Use the defined CSS custom properties for colors (`--accent`, `--warn`, `--info`, `--ok`, `--danger`, `--surface`, `--surface2`, `--text-dim`, etc.) — don't introduce new colors unless asked.

3. **Vanilla JS, no build step.** No bundler, no TypeScript, no React. ES5-compatible JS (`var`, `function`, no arrow functions in the older code paths). New code can use modern syntax but match the surrounding style.

4. **Cloudflare Worker constraints.** Worker code runs in V8 isolates — no Node.js APIs. Stream large response bodies through (`return new Response(r.body, ...)`) instead of buffering with `r.json()` to avoid CPU-time-exceeded errors on large Opus responses.

5. **PDF parsing** uses pdf.js loaded from CDN. The worker also handles audio extraction → WAV → AssemblyAI for video transcription.

6. **Demo data.** Some seeded demo rows exist for testing (e.g. the `4576+` line range with sample bid items). `DEMO_MODE` and `rowHasCurrentProjectProvenance()` gate whether these are visible — don't accidentally promote demo rows to real extractions.

## Deployment

- **Front-end:** static `BidTrace.html` (currently distributed as a zip). Could be hosted anywhere — Cloudflare Pages, Netlify, or served by the worker itself at its root route.
- **Worker:** deploy via Wrangler: `npx wrangler deploy` (uses `wrangler.toml`). Config is pinned to account `65ce4d6695638703c65edba36e718eb1` with `workers_dev = true` and custom routes for `bid-trace.com`.
- **CRITICAL — secrets after every deploy:** Cloudflare's versioning system clears secrets on each new code deployment. After every `wrangler deploy`, go to **Cloudflare dashboard → Workers & Pages → bidtrace-proxy → Settings → Variables and Secrets** and re-set `ANTHROPIC_API_KEY` and `ASSEMBLYAI_API_KEY` as Secret type. Verify with `curl https://bidtrace-proxy.dfried.workers.dev/ping` — both `has_anthropic_key` and `has_assemblyai_key` must be `true` before testing.

## Versioning notes

Iterating as `BidTrace_V3.zip`, `BidTrace_V4.zip`, `BidTrace_V5.zip` — recent change history:
- **V4:** QA Review redesigned as two-column sidebar + main layout; contractual items routed to a sidebar panel; `normalizeRowType` safety net added for verifier-added unpriced LS items.
- **V5:** Opus verifier prompt updated with explicit DO-NOT-ADD list for contractual obligations and tightened criteria for `missed_items` vs `method_notes_missed`.

Once on Claude Code with git, retire the zip-versioning and use commits + tags.
