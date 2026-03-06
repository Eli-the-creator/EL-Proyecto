# Project Instructions: AI Market Research Pipeline (v2)

## Who is the user
Former fullstack developer building a local MVP tool for personal market research. Communicate in technical language. The user speaks Russian but all code, comments, prompts, and LLM outputs should be in English.

## Project Overview
A local Next.js web application that replaces IdeaApe / GummySearch for Reddit-based market research. The tool collects Reddit discussions, analyzes them with AI, and generates structured customer personas with deep psychographic profiles, real quotes and source links.

**This is a personal tool, not a SaaS product. No commercial redistribution of Reddit data.**

**Cost per research: ~$3-4 per run.**

## Tech Stack
- **Framework:** Next.js 14+ (App Router) with JavaScript (not TypeScript)
- **UI:** shadcn/ui + Tailwind CSS
- **LLM:** Kimi K2.5 via DeepInfra (OpenAI-compatible API, 256K context window)
- **Search:** Serper.dev API
- **Reddit Data:** Apify actor `practicaltools/apify-reddit-api` ($2/1K items)
- **Data Storage:** JSON files in `data/` directory
- **Progress:** Server-Sent Events (SSE) — NOT polling

## Architecture — Pipeline Stages (MVP: Stage 0–4 + Discover Niche)

### Stage 0 — Query Optimization
- **Input:** User topic (required) + niche context (optional)
- **LLM:** Kimi K2.5 via DeepInfra
- **Output:** 6-8 Google search queries with `site:reddit.com`, each targeting a different angle
- **Format:** JSON array

### Stage 1 — Search (Serper.dev)
- **Input:** Queries from Stage 0
- **API:** POST https://google.serper.dev/search
- **Output:** 50-70 Reddit URLs → deduplicate → 40-60 unique URLs
- **Deduplication:** Normalize URLs, extract thread IDs, remove duplicates
- **Settings:** 10 results per query (default)

### Stage 1.5 — Reddit Data Fetch (Apify — Two-Phase)
- **Actor:** `practicaltools/apify-reddit-api`
- **Phase 1:** Fetch posts only (`skipComments: true`) for all ~50-60 URLs → get title, score, num_comments metadata. Cost: ~$0.12
- **Phase 2:** After Stage 2 filtering, fetch full threads with all comments for top 25-30 URLs only. Cost: ~$3.00
- **Output:** Normalized thread objects (post + ALL comments, no trimming)

### Stage 2 — Pre-filtering (No LLM)
- **Input:** Post metadata from Phase 1 of Stage 1.5
- **Logic:** JavaScript scoring — `keyword_relevance * 0.5 + engagement * 0.3 + recency * 0.2`
- **Engagement normalization:** Relative to median of current dataset (NOT hardcoded /500)
- **Filters:** score > 10, num_comments > 5
- **Output:** Top 25-30 thread URLs for Phase 2 fetch
- **No comment trimming** — all comments are kept for selected threads

### Stage 3 — Batch Analysis
- **Input:** 2-3 large batches of threads (leveraging Kimi's 256K context window)
- **LLM:** Kimi K2.5 (2-3 parallel calls)
- **Output:** Structured insights (pain_points, desires, behaviors, brand_mentions, emotional_signals)
- **Critical:** Every insight must have exact quote + thread_id + thread_url
- **Format:** Strict JSON with fallback parsing if LLM breaks format

### Stage 4 — Persona Synthesis + Psychographic Profile
- **Input:** All insights from Stage 3
- **LLM:** Kimi K2.5 (1 call)
- **Output:** 5-6 personas, each with:
  - Archetype name + description (150-200 words)
  - 5-7 Drivers, 5-7 Problems, 5-7 Opportunities (with quotes + source URLs)
  - Priority score (frequency * 0.6 + engagement * 0.4)
  - **Full psychographic profile:**
    - Core values and beliefs driving purchase decisions
    - Emotional triggers (what makes them act)
    - Fears and aspirations
    - Decision-making patterns (impulsive vs analytical, risk tolerance)
    - Social influences (whose opinion matters, communities they trust)
    - Information sources and research behavior
    - Price sensitivity and perceived value framework
  - All psychographic insights backed by real quotes and source URLs
- **Format:** Strict JSON with fallback parsing

### Stage 5 & 6 — OUT OF SCOPE FOR MVP

## App Structure — 5 Screens

### Screen 1: Input Form (Home)
- Text field: "Research Topic" (required)
- Text field: "Niche Context" (optional)
- Button: "Run Research"
- Below: list of past researches (history) with links to view results

### Screen 2: Progress / Loading (SSE-based)
- Shows which stage is currently running
- SSE-based: server pushes events to client via `/api/research/[id]/stream`
- Shows debug info: counts, timings, errors
- Example: "Found 67 Reddit URLs", "Filtered to 28 quality threads", "Analyzing batch 2/3"

### Screen 3: Results — Persona Cards
- Cards for each persona ordered by priority_score
- Each card shows: archetype name, description, Drivers, Problems, Opportunities
- **Psychographic profile section** with collapsible details
- Each Driver/Problem/Opportunity has supporting quote and source URL
- Below each persona card: list of all Reddit source URLs with "Copy All Links" button
- Global "Copy All Research Links" button (all unique URLs across all personas)

### Screen 4: Discover Niche
- Input field for broad topic (e.g. "fitness")
- LLM generates 10-15 sub-niches with descriptions
- Each sub-niche shows estimated discussion volume (via Serper)
- "Research this niche" button → redirects to Screen 1 with pre-filled topic

### Screen 5: History
- List of past researches stored in `data/` directory
- Each entry shows: topic, date, number of personas found
- Click to view full results (Screen 3)

### Placeholder Pages
- **Chat** — empty placeholder with "Coming Soon" message

## Data Storage Structure
```
data/
  {session_id}/
    meta.json              # topic, context, created_at, status, stage progress
    01_queries.json        # Stage 0 output
    02_search_results.json # Stage 1 output  
    03_reddit_urls.json    # Deduplicated URLs
    04_post_metadata.json  # Stage 1.5 Phase 1 output (posts only)
    05_filtered_urls.json  # Stage 2 output (top 25-30 URLs)
    06_full_threads.json   # Stage 1.5 Phase 2 output (posts + all comments)
    07_batch_results/      # Stage 3 output
      batch_01.json
      batch_02.json
      batch_03.json
    08_personas.json       # Stage 4 output (final result with psychographic profiles)
```

## API Routes Structure
```
app/
  api/
    research/
      route.js              # POST: start new research
      [id]/
        stream/route.js     # GET: SSE stream for progress
        route.js             # GET: get results
    history/
      route.js              # GET: list past researches
    discover/
      route.js              # POST: generate sub-niches for a topic
```

## Environment Variables
```
SERPER_API_KEY=xxx
DEEPINFRA_API_KEY=xxx
DEEPINFRA_BASE_URL=https://api.deepinfra.com/v1/openai
DEEPINFRA_MODEL=moonshotai/Kimi-K2.5
APIFY_API_TOKEN=xxx
APIFY_REDDIT_ACTOR=practicaltools/apify-reddit-api
DATA_DIR=./data
```

## LLM Call Configuration
- **Base URL:** https://api.deepinfra.com/v1/openai/chat/completions
- **Model:** moonshotai/Kimi-K2.5
- **Context window:** 256K tokens
- **API format:** OpenAI-compatible (use standard fetch, no SDK needed)
- **max_tokens:** Control to prevent verbose output
- **temperature:** 0.3 for analysis stages (consistency), 0.7 for query optimization and niche discovery (creativity)
- **JSON output:** Request JSON in system prompt + validate/fallback parse response

## JSON Fallback Parsing Strategy
1. Try `JSON.parse(response)` first
2. If fails, try to extract JSON from ```json ... ``` blocks
3. If fails, try to find first `[` or `{` and last `]` or `}` and parse that substring
4. If all fail, log error and retry the LLM call once with stricter prompt
5. If retry fails, save raw response for debugging and skip to next stage with partial data

## Apify Configuration
- **Actor:** `practicaltools/apify-reddit-api`
- **API:** Use Apify REST API (run-sync-get-dataset-items endpoint)
- **Phase 1 input:** `{ "startUrls": [...], "skipComments": true }`
- **Phase 2 input:** `{ "startUrls": [...], "skipComments": false }`
- **Pricing:** $2 per 1,000 items, first 1,000/month free
- **Error handling:** Retry once on timeout, log and continue on failure

## URL Deduplication
```javascript
function normalizeRedditUrl(url) {
  const match = url.match(/reddit\.com\/r\/(\w+)\/comments\/(\w+)/);
  if (!match) return null;
  return { 
    subreddit: match[1], 
    threadId: match[2],
    normalized: `https://www.reddit.com/r/${match[1]}/comments/${match[2]}/`
  };
}

function deduplicateUrls(urls) {
  const seen = new Set();
  return urls.filter(url => {
    const parsed = normalizeRedditUrl(url);
    if (!parsed || seen.has(parsed.threadId)) return false;
    seen.add(parsed.threadId);
    return true;
  });
}
```

## Code Style Rules
- This is an MVP — working code over perfect code
- Keep it simple, no over-engineering
- Inline comments for non-obvious logic
- No separate CSS files — use Tailwind classes
- shadcn/ui for UI components
- All LLM prompts stored in a separate `lib/prompts.js` file for easy iteration
- All API helper functions in `lib/` directory
- Error handling: log errors, skip failed items, continue pipeline

## What NOT to build
- No authentication/login
- No database (use JSON files)
- No PDF export
- No deployment config (local only)
- No tests (MVP)
- No Docker for MVP (run with `npm run dev`) — Docker/VPN is a future enhancement
- No Reddit OAuth spoofing — use Apify only
- No Kubernetes
