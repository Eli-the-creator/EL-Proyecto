# Tech Spec: AI Market Research Pipeline (Next.js MVP) — v2

## Pipeline Flow Diagram
```
[User Input] → Stage 0 (LLM: Query Optimization)
    → Stage 1 (Serper.dev: Search)
    → Stage 1.5 Phase 1 (Apify: Fetch Post Metadata)
    → Stage 2 (JS: Pre-filtering + Scoring)
    → Stage 1.5 Phase 2 (Apify: Fetch Full Threads)
    → Stage 3 (LLM: Batch Analysis, 2-3 batches)
    → Stage 4 (LLM: Persona Synthesis + Psychographic Profile)
    → [Display Results + Save to disk]
```

---

## Stage 0 — Query Optimization

### Purpose
Convert user's raw topic into 6-8 optimized Google search queries targeting different angles of Reddit discussions.

### Input
```json
{
  "topic": "period leak underwear",
  "context": "women's hygiene products"
}
```

### LLM Prompt
```
SYSTEM:
You are a search query optimization expert for Reddit market research.

Your task: convert a research topic into 6-8 Google search queries that will find the most valuable Reddit discussions for consumer research.

Rules:
- Every query MUST include "site:reddit.com"
- Each query targets a DIFFERENT angle (see list below)
- Use natural language that real Reddit users would use
- Include synonyms and variations of key terms
- Keep queries specific — generic queries return irrelevant results
- Do NOT use boolean operators (AND/OR) — Google handles them poorly with site: operator

Required angles (pick 6-8):
1. Complaints, frustrations, problems
2. Positive experiences, recommendations, "what works"
3. Brand/product comparisons ("X vs Y")
4. Specific use cases, situations, scenarios
5. Price, value, budget discussions
6. Tips, hacks, workarounds, advice
7. Alternatives and substitutes
8. Emotional experiences, stories, rants

Respond with ONLY a JSON array. No explanation, no markdown.

Example response format:
[
  {
    "angle": "complaints",
    "query": "site:reddit.com period underwear leaking problems frustrated",
    "purpose": "Find frustrations and pain points"
  }
]

USER:
Topic: "{{topic}}"
{{#if context}}Niche context: "{{context}}"{{/if}}

Generate optimized search queries.
```

### LLM Settings
- temperature: 0.7
- max_tokens: 2000

### Validation
- Must be valid JSON array
- Each item must have `angle`, `query`, `purpose` fields
- Each `query` must contain "site:reddit.com"
- Array length: 5-8 items

---

## Stage 1 — Search (Serper.dev)

### API Call
```
POST https://google.serper.dev/search
Headers: { "X-API-KEY": "{{SERPER_API_KEY}}", "Content-Type": "application/json" }
Body: { "q": "{{query}}", "num": 10 }
```

### Response Processing
1. Extract `link` from each item in `organic` array
2. Filter: keep only URLs matching `reddit.com/r/*/comments/*`
3. Normalize URLs and extract thread IDs
4. Deduplicate by thread ID across all query results
5. Save deduplicated URL list

### Output
Array of 40-60 unique Reddit URLs.

---

## Stage 1.5 — Reddit Data Fetch (Apify, Two-Phase)

### Phase 1: Post Metadata Only

**Purpose:** Get lightweight metadata for all URLs to enable smart filtering before paying for comments.

**API Call:**
```javascript
const response = await fetch(
  `https://api.apify.com/v2/acts/practicaltools~apify-reddit-api/run-sync-get-dataset-items?token=${APIFY_TOKEN}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      startUrls: urls.map(url => ({ url })),
      skipComments: true,
      skipCommunity: true,
      skipUserPosts: true
    })
  }
);
```

**Output:** Array of post objects with title, score, num_comments, url, etc.
**Cost:** ~60 items × $0.002 = ~$0.12

### Phase 2: Full Threads with Comments

**Purpose:** Fetch complete thread data (post + ALL comments) for filtered URLs only.

**Input:** Top 25-30 URLs from Stage 2 filtering.

**API Call:** Same as Phase 1 but with `skipComments: false`.

**Output:** Full thread objects with all comments.
**Cost:** ~25-30 posts + ~1500 comments = ~$3.00

### Normalized Thread Format (after Phase 2)
```json
{
  "thread_id": "abc123",
  "subreddit": "menstrualcups",
  "title": "Best period underwear for heavy flow?",
  "body": "I've been looking for...",
  "author": "user123",
  "score": 234,
  "num_comments": 89,
  "created_utc": "2024-03-01T12:00:00.000Z",
  "url": "https://www.reddit.com/r/menstrualcups/comments/abc123/",
  "comments": [
    {
      "author": "commenter1",
      "body": "I tried Thinx and they leaked on the first day...",
      "score": 45
    }
  ]
}
```

### Comment Normalization Rules
- Flatten nested comment tree into flat array
- Skip comments by [deleted] or AutoModerator
- Skip comments with body "[removed]" or "[deleted]"
- Skip comments shorter than 20 characters
- Keep ALL remaining comments (no trimming)

---

## Stage 2 — Pre-filtering (JavaScript, No LLM)

### Purpose
Score and filter posts by relevance, engagement, and recency. Select top 25-30 for full comment fetch.

### Scoring Formula
```javascript
function scoreThread(thread, keywords) {
  const text = [thread.title, thread.body || ''].join(' ').toLowerCase();
  const keywordHits = keywords.filter(kw => text.includes(kw.toLowerCase()));
  const relevance = keywordHits.length / keywords.length;

  // Engagement normalized to MEDIAN of current dataset
  const allEngagements = allThreads.map(t => t.score + t.num_comments * 2);
  const median = getMedian(allEngagements);
  const engagement = Math.min(1, (thread.score + thread.num_comments * 2) / (median * 2));

  const ageInDays = (Date.now() - new Date(thread.created_utc).getTime()) / 86400000;
  const recency = Math.max(0, 1 - ageInDays / 730);

  return relevance * 0.5 + engagement * 0.3 + recency * 0.2;
}
```

### Filtering Steps
1. Remove threads with score < 10
2. Remove threads with num_comments < 5
3. Score all remaining threads
4. Sort by composite score descending
5. Take top 30 threads (or all if fewer)

---

## Stage 3 — Batch Analysis

### Batching Strategy
- Split 25-30 threads into 2-3 batches (leveraging Kimi's 256K context)
- Run batches in parallel (2-3 concurrent LLM calls)
- Each batch produces its own insights array
- Aggregate all batch results at the end

### LLM Prompt
```
SYSTEM:
You are a market research analyst specializing in consumer behavior analysis and psychographic profiling from online discussions.

Analyze the provided Reddit threads and extract structured consumer insights.

CRITICAL RULES:
- ONLY use information explicitly stated in the threads
- NEVER invent, assume, or hallucinate data
- Every insight MUST include an exact quote copied from the source (not paraphrased)
- Every insight MUST reference thread_id and thread_url
- Classify each insight into exactly one category
- Pay special attention to EMOTIONAL language, VALUES expressed, and DECISION-MAKING patterns

CATEGORIES:
1. pain_point — frustrations, complaints, problems, things that don't work
2. desire — wants, wishes, goals, "I wish...", "I want..."
3. behavior — habits, routines, purchasing patterns, decision-making
4. brand_mention — specific products/brands discussed (positive, negative, or neutral)
5. emotional_signal — fears, anxieties, hopes, excitement, anger, relief
6. value_signal — core values expressed (health, convenience, price, sustainability, etc.)
7. social_influence — mentions of recommendations from friends, family, influencers, communities

For each insight extract:
- type: one of the categories above
- description: 1-2 sentence summary of the insight
- severity: 1-5 (how intense/important this is based on context)
- quote: EXACT text copied from the post or comment (not paraphrased)
- author: Reddit username of the person quoted
- thread_id: ID of the source thread
- thread_url: full URL of the source thread
- thread_title: title of the source thread

Extract 3-10 insights per thread depending on content richness.
Skip off-topic comments, jokes, and promotional content.
Focus on genuine consumer voice.

Respond with ONLY valid JSON. No markdown, no explanation.

USER:
Analyze these Reddit threads:
{{batch_threads_json}}

Response format:
{
  "insights": [...]
}
```

### LLM Settings
- temperature: 0.3
- max_tokens: 8000 per batch

---

## Stage 4 — Persona Synthesis + Psychographic Profile

### LLM Prompt
```
SYSTEM:
You are a senior market research strategist and consumer psychologist creating deep customer personas from real consumer data.

Based on the extracted insights from Reddit discussions, identify and synthesize 5-6 distinct customer personas with full psychographic profiles.

CRITICAL RULES:
- Every persona must be grounded in the data provided — do NOT invent or assume
- Every driver, problem, and opportunity must be traceable to real insights with quotes
- Personas must be meaningfully different from each other
- Quotes must be real (from the provided data), never fabricated
- Order personas by priority_score descending
- Psychographic insights must be DERIVED from actual quotes, not generic assumptions

PRIORITY SCORE CALCULATION:
- frequency_weight (0.6): How often this persona's patterns appear across the data
- engagement_weight (0.4): Average engagement of source threads for this persona
- Final score: 1-100 scale

FOR EACH PERSONA PROVIDE:

1. archetype_name: A memorable 2-4 word name

2. description: 150-200 word narrative about this person

3. priority_score: 1-100

4. drivers: Array of 5-7 objects. What motivates this persona.
   Each: { "text": "...", "quote": "exact quote", "source_url": "reddit URL" }

5. problems: Array of 5-7 objects. Barriers, frustrations, pain points.
   Each: { "text": "...", "quote": "exact quote", "source_url": "reddit URL" }

6. opportunities: Array of 5-7 objects. How brands can better serve this persona.
   Each: { "text": "...", "quote": "exact quote", "source_url": "reddit URL" }

7. psychographic_profile: {
     "core_values": [
       { "value": "...", "evidence": "exact quote", "source_url": "..." }
     ],
     "emotional_triggers": [
       { "trigger": "...", "response": "what they do/feel", "quote": "...", "source_url": "..." }
     ],
     "fears_and_anxieties": [
       { "fear": "...", "quote": "...", "source_url": "..." }
     ],
     "aspirations": [
       { "aspiration": "...", "quote": "...", "source_url": "..." }
     ],
     "decision_making_pattern": {
       "style": "impulsive|analytical|social-proof-driven|price-driven|quality-driven",
       "description": "...",
       "evidence": "exact quote",
       "source_url": "..."
     },
     "social_influences": [
       { "influence": "...", "quote": "...", "source_url": "..." }
     ],
     "information_sources": ["where they research before buying"],
     "price_sensitivity": {
       "level": "low|medium|high",
       "description": "...",
       "evidence": "exact quote",
       "source_url": "..."
     }
   }

8. source_threads: Array of ALL Reddit URLs that contributed insights to this persona.

Respond with ONLY valid JSON. No markdown, no explanation.

USER:
Research topic: "{{topic}}"
Total threads analyzed: {{total_threads}}
Total insights extracted: {{total_insights}}

Insights data:
{{all_insights_json}}

Response format:
{
  "personas": [...]
}
```

### LLM Settings
- temperature: 0.3
- max_tokens: 12000

### Output Validation
- Must be valid JSON with `personas` array
- Each persona must have all required fields
- `drivers`, `problems`, `opportunities` arrays: 5-7 items each
- Each item must have `text`, `quote`, `source_url`
- `psychographic_profile` must have all sub-fields
- `source_threads` must be non-empty array
- `priority_score` must be 1-100

---

## Discover Niche Feature

### Purpose
Help user narrow down a broad topic into specific research-worthy niches.

### API: POST /api/discover

### LLM Prompt
```
SYSTEM:
You are a niche market research expert. Given a broad topic, identify 10-15 specific sub-niches that have strong potential for consumer research.

For each sub-niche provide:
- name: Short, specific niche name (3-8 words)
- description: 1-2 sentences explaining the niche and why it's interesting for research
- search_query: A Google query to estimate discussion volume (include "site:reddit.com")
- estimated_demand: "high" | "medium" | "low" based on your knowledge
- pain_level: 1-5 (how much pain/frustration exists in this niche)

Focus on niches where:
- Real people have genuine problems to solve
- There's enough discussion volume on Reddit
- Commercial opportunity exists (people are willing to pay for solutions)

Respond with ONLY valid JSON. No markdown, no explanation.

USER:
Broad topic: "{{topic}}"

Response format:
{
  "niches": [...]
}
```

### Enrichment
After LLM generates niches, optionally run 2-3 Serper queries to validate discussion volume and add `estimated_posts` count to each niche.

---

## SSE Progress Streaming

### Endpoint: GET /api/research/[id]/stream

### Event Format
```
data: {"stage":"stage_0","status":"running","detail":"Generating search queries..."}

data: {"stage":"stage_1","status":"running","detail":"Searching query 3/7"}

data: {"stage":"stage_1_5_p1","status":"running","detail":"Fetching post metadata for 56 URLs"}

data: {"stage":"stage_2","status":"completed","detail":"Filtered to 28 quality threads"}

data: {"stage":"stage_1_5_p2","status":"running","detail":"Fetching full threads with comments (28 URLs)"}

data: {"stage":"stage_3","status":"running","detail":"Analyzing batch 2/3"}

data: {"stage":"stage_4","status":"running","detail":"Synthesizing personas with psychographic profiles"}

data: {"stage":"completed","status":"completed","detail":"Generated 6 personas","personas_count":6}

data: {"stage":"error","status":"error","detail":"Apify timeout on phase 2","error":"..."}
```

### Client Implementation
```javascript
const eventSource = new EventSource(`/api/research/${id}/stream`);
eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateProgress(data);
};
```

---

## Pipeline Orchestration

### Session ID Format
```javascript
const sessionId = `${Date.now()}_${crypto.randomBytes(2).toString('hex')}_${topic.toLowerCase().replace(/\s+/g, '_').slice(0, 30)}`;
```

### Background Execution
Fire-and-forget from POST /api/research handler. Pipeline runs in background, writes progress to meta.json and pushes SSE events. This works for local `npm run dev`.

---

## Error Handling Strategy

### LLM Errors
- JSON parse failure → fallback parsing → retry once → save raw response
- Timeout → retry once
- Rate limit (429) → wait 30 seconds, retry

### Apify Errors
- Timeout → retry once with same input
- Rate limit → wait and retry
- Empty results → log warning, continue with available data
- Actor error → log, skip phase, report in SSE

### Serper Errors
- 401 → invalid API key, stop pipeline with error
- 429 → wait 10 seconds, retry
- Empty results → log warning, continue

### General Principle
Pipeline should be resilient. Skip failed items, continue with available data. Only stop for critical errors (invalid API keys, zero data collected).

---

## File Structure
```
market-research-app/
├── app/
│   ├── layout.js
│   ├── page.js                    # Home: input form + history
│   ├── research/
│   │   └── [id]/
│   │       └── page.js            # Results page (SSE progress + persona display)
│   ├── discover/
│   │   └── page.js                # Discover Niche page
│   ├── chat/
│   │   └── page.js                # Placeholder: "Coming Soon"
│   └── api/
│       ├── research/
│       │   ├── route.js           # POST: start research
│       │   └── [id]/
│       │       ├── route.js       # GET: full results
│       │       └── stream/
│       │           └── route.js   # GET: SSE progress stream
│       ├── history/
│       │   └── route.js           # GET: list past researches
│       └── discover/
│           └── route.js           # POST: generate sub-niches
├── lib/
│   ├── prompts.js                 # All LLM prompts
│   ├── llm.js                     # DeepInfra API helper
│   ├── serper.js                  # Serper.dev API helper
│   ├── apify.js                   # Apify API helper (two-phase fetch)
│   ├── scoring.js                 # Pre-filtering logic
│   ├── pipeline.js                # Main pipeline orchestrator
│   ├── storage.js                 # JSON file read/write helpers
│   └── sse.js                     # SSE helper for progress streaming
├── components/
│   ├── ResearchForm.jsx           # Input form component
│   ├── ProgressTracker.jsx        # SSE-based loading/progress display
│   ├── PersonaCard.jsx            # Single persona card with psychographic profile
│   ├── PersonaList.jsx            # List of persona cards
│   ├── PsychographicProfile.jsx   # Psychographic profile display
│   ├── HistoryList.jsx            # Past researches list
│   ├── NicheDiscovery.jsx         # Discover Niche UI
│   ├── CopyLinksButton.jsx        # Copy URLs to clipboard
│   └── Navigation.jsx             # App navigation with page links
├── data/                          # Research data (gitignored)
├── .env.local                     # API keys
├── package.json
└── README.md
```
