# Claude Code Prompts — Ready to Use

## How to Use This File

1. Start Claude Code in your project directory
2. Before EACH phase, give Claude Code access to the project docs:
   - `PROJECT_INSTRUCTIONS.md`
   - `TECH_SPEC.md`
   - `IMPLEMENTATION_PLAN.md`
3. Copy-paste the prompt for the current phase
4. After each phase, TEST manually before moving to next
5. If something breaks, share the error with Claude Code and ask to fix

## Important: Context Setup

At the START of each Claude Code session, run this first:

```
Read these project files and understand the full context before doing anything:
- PROJECT_INSTRUCTIONS.md
- TECH_SPEC.md  
- IMPLEMENTATION_PLAN.md

Confirm you understand the project architecture, tech stack, and current phase.
```

---

## Phase 1 — Skeleton + Plumbing

```
Build Phase 1 from the IMPLEMENTATION_PLAN.md.

Create a new Next.js project called "market-research-app" with App Router, JavaScript (not TypeScript), and Tailwind.

Set up shadcn/ui and add these components: button, card, input, badge, textarea, separator, toast, tabs, collapsible, progress.

Create these files:
- .env.local with placeholder keys (SERPER_API_KEY, DEEPINFRA_API_KEY, APIFY_API_TOKEN, DEEPINFRA_BASE_URL, DEEPINFRA_MODEL, DATA_DIR)
- lib/storage.js — createSession, saveStageData, loadStageData, getSessionStatus, listSessions (JSON file storage in data/ directory)
- lib/llm.js — callKimi(systemPrompt, userPrompt, options) with the 5-step JSON fallback parsing from TECH_SPEC
- lib/sse.js — SSE helper for streaming progress events
- API routes: POST /api/research (stub), GET /api/research/[id]/stream (SSE), GET /api/research/[id] (results), GET /api/history
- components/Navigation.jsx — sidebar with links: Research, Discover Niche, Chat
- app/layout.js with Navigation
- app/page.js — input form (topic + context fields) + history list
- app/research/[id]/page.js — SSE progress display + placeholder results
- app/discover/page.js — placeholder with input field
- app/chat/page.js — "Coming Soon" placeholder

Use shadcn/ui components and Tailwind. No separate CSS files. Session ID format: timestamp + random hex + topic slug.

After creating, verify the app starts with npm run dev.
```

---

## Phase 2 — Stage 0 + Stage 1

```
Build Phase 2. The project skeleton from Phase 1 exists.

Implement:
1. lib/prompts.js — Add the Stage 0 query optimization prompt from TECH_SPEC.md
2. Stage 0 in lib/pipeline.js: call Kimi with the prompt → parse JSON → save to 01_queries.json
3. lib/serper.js — searchGoogle(query) function that calls Serper.dev API
4. Stage 1: run all queries through Serper → extract Reddit URLs → filter only reddit.com/r/*/comments/* URLs
5. URL deduplication: normalize URLs, extract thread IDs, deduplicate → save to 02_search_results.json and 03_reddit_urls.json
6. Wire up SSE progress events for both stages
7. Update the POST /api/research route to actually start the pipeline (fire-and-forget)

Follow the prompts and API formats exactly as specified in TECH_SPEC.md.
Test: submitting "period leak underwear" should generate 6-8 queries and find 40-60 unique Reddit URLs.
```

---

## Phase 3 — Reddit Data + Filtering

```
Build Phase 3. Stages 0-1 are working.

Implement the two-phase Apify integration and filtering:

1. lib/apify.js with two functions:
   - fetchPostMetadata(urls) — calls Apify with skipComments:true, returns post metadata
   - fetchFullThreads(urls) — calls Apify with skipComments:false, returns full threads
   Use the Apify run-sync-get-dataset-items endpoint as shown in TECH_SPEC.md.

2. Stage 1.5 Phase 1: fetch post metadata for all deduplicated URLs → save to 04_post_metadata.json

3. lib/scoring.js — scoreThread function with:
   - keyword_relevance * 0.5 + engagement * 0.3 + recency * 0.2
   - Engagement normalized to MEDIAN of current dataset (not hardcoded)
   - Filters: score > 10, num_comments > 5

4. Stage 2: score and filter posts → select top 25-30 → save filtered URLs to 05_filtered_urls.json

5. Stage 1.5 Phase 2: fetch full threads with all comments for filtered URLs only → normalize comments (flatten, skip deleted/automod/short) → save to 06_full_threads.json

6. Wire up SSE events for all sub-stages with counts and progress

Test: should go from 50+ URLs → metadata → filter to 25-30 → full threads with all comments
```

---

## Phase 4 — LLM Analysis + Personas

```
Build Phase 4. Data collection pipeline (Stages 0-2) is working.

Implement:

1. Add Stage 3 prompt to lib/prompts.js — the batch analysis prompt from TECH_SPEC.md. Note the 7 insight categories including value_signal and social_influence.

2. Stage 3 in pipeline.js:
   - Split threads into 2-3 batches (aim for roughly equal token count per batch)
   - Run batches in parallel with Promise.allSettled (2-3 concurrent calls)
   - Aggregate all insights from all batches
   - Save batch results to 07_batch_results/ directory

3. Add Stage 4 prompt to lib/prompts.js — the persona synthesis prompt with FULL psychographic profile structure from TECH_SPEC.md. This is the most important prompt — copy it exactly.

4. Stage 4: single LLM call with all aggregated insights → parse personas with psychographic profiles → validate output structure → save to 08_personas.json

5. Complete pipeline orchestrator: chain all stages (0 → 1 → 1.5p1 → 2 → 1.5p2 → 3 → 4), handle errors at each stage, push SSE events throughout.

6. max_tokens for Stage 3: 8000 per batch. max_tokens for Stage 4: 12000.

Test: full pipeline end-to-end with "period leak underwear". Should produce 5-6 personas with psychographic profiles.
```

---

## Phase 5 — Results UI

```
Build Phase 5. Full pipeline works end-to-end.

Build the results display UI:

1. PersonaCard component:
   - Archetype name as card header with priority score badge
   - Description paragraph
   - Three tabs: Drivers, Problems, Opportunities
   - Each item shows: text, collapsible quote with source link
   - "Copy All Links" button per persona (copies all source URLs to clipboard)

2. PsychographicProfile component (collapsible section inside PersonaCard):
   - Core Values — list with evidence quotes
   - Emotional Triggers — trigger + response + quote
   - Fears & Anxieties — with quotes
   - Aspirations — with quotes
   - Decision-Making Pattern — style badge + description + evidence
   - Social Influences — with quotes
   - Information Sources — simple list
   - Price Sensitivity — level badge + description + evidence

3. ProgressTracker component:
   - SSE-based, shows current stage with animated indicator
   - Shows detail text from SSE events (counts, batch progress)
   - Debug section (collapsible) showing raw event log

4. Update app/research/[id]/page.js:
   - While running: show ProgressTracker
   - When complete: show PersonaList with all PersonaCards
   - Global "Copy All Research Links" button at top

5. HistoryList component on home page — shows past researches with topic, date, persona count

6. Responsive layout — works on desktop and tablet

Use shadcn/ui components (Card, Badge, Tabs, Collapsible, Button, Progress). Tailwind for styling.
```

---

## Phase 6 — Discover Niche + Polish

```
Build Phase 6. Main research flow is complete.

1. Add Discover Niche prompt to lib/prompts.js (from TECH_SPEC.md)

2. Create app/api/discover/route.js:
   - POST endpoint, receives { topic }
   - Calls Kimi with discover prompt → parse JSON
   - Optionally validate 2-3 niches with Serper to add discussion volume estimate
   - Return niches array

3. Build NicheDiscovery component on app/discover/page.js:
   - Input field for broad topic + "Discover" button
   - Loading state while LLM generates
   - Grid of niche cards, each showing:
     - Niche name (bold)
     - Description
     - Estimated demand badge (high/medium/low)
     - Pain level indicator (1-5)
     - "Research this niche" button
   - "Research this niche" navigates to /?topic={niche_name}

4. Update app/page.js to accept ?topic= query parameter and pre-fill the form

5. Polish:
   - Error states for all pages (API failures, empty results)
   - Loading states (skeletons or spinners)
   - Empty states ("No researches yet")
   - Toast notifications for copy-to-clipboard actions
   - Mobile responsiveness check

Test: enter "fitness" in Discover → see 10-15 sub-niches → click one → land on research page with pre-filled topic
```

---

## Debugging Tips for Claude Code

If pipeline fails at a specific stage:
```
The pipeline fails at Stage X with error: [paste error].
Check the saved data in data/{session_id}/ to see what the previous stages produced.
Fix the issue and make the pipeline resilient — it should skip failed items and continue.
```

If LLM returns bad JSON:
```
Kimi is returning malformed JSON for Stage X. Here's the raw response: [paste].
Make sure the 5-step JSON fallback parsing in lib/llm.js handles this case.
```

If Apify returns unexpected data:
```
Apify returned this data structure: [paste sample].
Update lib/apify.js to normalize this into our expected thread format from TECH_SPEC.md.
```
