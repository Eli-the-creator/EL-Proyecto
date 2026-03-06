# Implementation Plan: Build Order (v2)

## Phase 1 — Skeleton + Plumbing
Set up the project foundation before any business logic.

### Tasks:
1. `npx create-next-app@latest market-research-app` (App Router, JavaScript, Tailwind, no src/ directory)
2. Install dependencies: `shadcn/ui` (init + add button, card, input, badge, textarea, separator, toast, tabs, collapsible, progress)
3. Create `.env.local` with placeholder API keys (SERPER_API_KEY, DEEPINFRA_API_KEY, APIFY_API_TOKEN)
4. Create `lib/storage.js` — helper functions: createSession, saveStageData, loadStageData, getSessionStatus, listSessions
5. Create `lib/llm.js` — helper function: callKimi(systemPrompt, userPrompt, options) with JSON fallback parsing (5-step strategy)
6. Create `lib/sse.js` — SSE helper for progress streaming
7. Create `app/api/research/route.js` — POST endpoint that creates session and starts pipeline (stub)
8. Create `app/api/research/[id]/stream/route.js` — GET endpoint for SSE progress stream
9. Create `app/api/research/[id]/route.js` — GET endpoint that reads final results
10. Create `app/api/history/route.js` — GET endpoint that lists sessions from data/ directory
11. Create `components/Navigation.jsx` — sidebar/header with links: Research, Discover Niche, Chat
12. Create `app/layout.js` with Navigation component
13. Create basic `app/page.js` with input form and history list
14. Create `app/research/[id]/page.js` with SSE progress + placeholder results display
15. Create `app/discover/page.js` — placeholder with input field
16. Create `app/chat/page.js` — "Coming Soon" placeholder

**Test:** form submits → session created → SSE stream works → history shows entry → navigation between pages works

---

## Phase 2 — Stage 0 + Stage 1 + Deduplication
Search queries generation, execution, and URL deduplication.

### Tasks:
1. Create `lib/prompts.js` with Stage 0 prompt
2. Implement Stage 0 in `lib/pipeline.js`: call Kimi → parse JSON → save queries
3. Create `lib/serper.js` — helper function: searchGoogle(query) → array of organic results
4. Implement Stage 1: run all queries through Serper → extract Reddit URLs
5. Implement URL deduplication: normalize URLs, extract thread IDs, remove duplicates → save
6. Wire up SSE progress events for Stage 0 and Stage 1

**Test with:** topic "period leak underwear" — should produce 6-8 queries and 40-60 unique Reddit URLs

---

## Phase 3 — Stage 1.5 Phase 1 + Stage 2 + Stage 1.5 Phase 2
Two-phase Reddit data collection with smart filtering in between.

### Tasks:
1. Create `lib/apify.js` — helper functions: fetchPostMetadata(urls), fetchFullThreads(urls)
2. Implement Stage 1.5 Phase 1: call Apify with skipComments=true → get post metadata → save
3. Create `lib/scoring.js` — scoreThread(thread, keywords) with median-based engagement normalization
4. Implement Stage 2: score → filter → sort → select top 25-30 URLs → save filtered list
5. Implement Stage 1.5 Phase 2: call Apify with skipComments=false for filtered URLs → normalize comments → save
6. Wire up SSE progress events for all sub-stages

**Test:** Should go from 50+ URLs → metadata for all → filter to 25-30 → full threads with all comments

---

## Phase 4 — Stage 3 + Stage 4
LLM analysis with psychographic profiling.

### Tasks:
1. Add Stage 3 prompt to `lib/prompts.js` (with value_signal and social_influence categories)
2. Implement Stage 3: split into 2-3 batches → parallel LLM calls → aggregate insights → save
3. Add Stage 4 prompt to `lib/prompts.js` (with full psychographic profile structure)
4. Implement Stage 4: single LLM call with all insights → parse personas with psychographic profiles → save
5. Complete the pipeline orchestrator: chain all stages, handle errors, push SSE events
6. Wire up SSE events for Stage 3 and Stage 4

**Test:** Full pipeline end-to-end with "period leak underwear"

---

## Phase 5 — UI: Results Display
Make results display useful with psychographic profiles.

### Tasks:
1. Build `PersonaCard` component with Drivers/Problems/Opportunities sections
2. Each section item shows: text + quote (collapsible) + source link
3. Build `PsychographicProfile` component with collapsible sections:
   - Core Values, Emotional Triggers, Fears & Anxieties, Aspirations
   - Decision-Making Pattern, Social Influences, Information Sources, Price Sensitivity
4. Each psychographic insight shows quote + source link
5. "Copy All Links" button per persona
6. "Copy All Research Links" button (global, all unique URLs)
7. `ProgressTracker` component with SSE-based stage indicators and debug info
8. `HistoryList` component with links to past results
9. Basic responsive layout

**Test:** Full user flow from input to browsing results, psychographic profiles, and copying links

---

## Phase 6 — Discover Niche + Polish
Niche discovery feature and final touches.

### Tasks:
1. Add Discover Niche prompt to `lib/prompts.js`
2. Create `app/api/discover/route.js` — POST endpoint
3. Implement: LLM generates sub-niches → optional Serper validation → return results
4. Build `NicheDiscovery` component with niche cards
5. Each niche card: name, description, estimated demand, pain level, "Research this niche" button
6. "Research this niche" → navigate to home page with pre-filled topic
7. Final UI polish: error states, loading states, empty states
8. Mobile responsiveness check

**Test:** Enter "fitness" → see 10-15 sub-niches → click "Research this niche" → land on home with topic pre-filled

---

## Cost Summary Per Research Run
- Serper.dev (6-8 queries): ~$0.05
- Apify Phase 1 (60 posts): ~$0.12
- Apify Phase 2 (25-30 threads + comments): ~$3.00
- Kimi K2.5 Stage 0 (1 call): ~$0.01
- Kimi K2.5 Stage 3 (2-3 calls): ~$0.10
- Kimi K2.5 Stage 4 (1 call): ~$0.05
- **Total: ~$3.30 per research**

## Estimated Build Time
- Phase 1: 2-3 hours
- Phase 2: 2-3 hours
- Phase 3: 3-4 hours
- Phase 4: 3-4 hours
- Phase 5: 3-4 hours
- Phase 6: 2-3 hours
- **Total: ~15-21 hours of focused work**
