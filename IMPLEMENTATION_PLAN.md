# Beyond the Camera — Implementation Plan

> **Scope:** MVP / Phase 1 prototype — two working vertical slices (3 political scenarios + 1 journalistic scenario), local infrastructure, no auth, text-only I/O.

---

## Table of Contents

1. [Phase 0 — Repo & Toolchain Setup](#phase-0--repo--toolchain-setup)
2. [Phase 1 — Backend Foundation](#phase-1--backend-foundation)
3. [Phase 2 — Topic Intelligence Pipeline](#phase-2--topic-intelligence-pipeline)
4. [Phase 3 — Simulation Engine](#phase-3--simulation-engine)
5. [Phase 4 — Feedback & Coaching Module](#phase-4--feedback--coaching-module)
6. [Phase 5 — Frontend (Next.js)](#phase-5--frontend-nextjs)
7. [Phase 6 — Integration & End-to-End Slice](#phase-6--integration--end-to-end-slice)
8. [Phase 7 — Polish & Handoff to Phase 2](#phase-7--polish--handoff-to-phase-2)
9. [Tech Decisions Recap](#tech-decisions-recap)
10. [Out of Scope (Phase 1)](#out-of-scope-phase-1)

---

## Phase 0 — Repo & Toolchain Setup

**Goal:** Every developer can clone the repo and spin up the full local dev environment in under 15 minutes.

### Steps

1. **Scaffold project structure**
   ```
   beyond_the_camera_ag/
   ├── backend/
   │   ├── app/
   │   │   ├── api/
   │   │   ├── core/
   │   │   ├── models/
   │   │   ├── services/
   │   │   └── main.py
   │   ├── pyproject.toml
   │   └── .env.example
   ├── frontend/
   │   ├── src/
   │   │   ├── app/
   │   │   ├── components/
   │   │   └── lib/
   │   └── package.json
   ├── scripts/
   ├── docker-compose.yml
   └── README.md
   ```

2. **Backend toolchain**
   - Python `3.12` via `pyenv`
   - Package manager: `uv` + `pyproject.toml`
   - Dependencies: `fastapi`, `uvicorn[standard]`, `sqlalchemy`, `alembic`, `psycopg2-binary`, `redis`, `openai`, `pydantic-settings`, `httpx`, `python-dotenv`
   - Dev dependencies: `pytest`, `httpx` (async test client), `pytest-asyncio`

3. **Frontend toolchain**
   - Node `22 LTS` via `nvm`; pin version in `.nvmrc`
   - Bootstrap: `npx create-next-app@latest frontend --ts --app --eslint --no-tailwind`
   - CSS: Vanilla CSS modules
   - API client: `fetch` wrapper in `src/lib/api.ts`

4. **Docker Compose** — define services:
   - `postgres:16` — port `5432`, `POSTGRES_PASSWORD=secret`
   - `redis:7` — port `6379`
   - (optional) `langfuse` for LLM observability

5. **Environment files**
   - `backend/.env.example` with all required keys:
     ```
     AZURE_OPENAI_ENDPOINT=
     AZURE_OPENAI_API_KEY=
     AZURE_OPENAI_DEPLOYMENT_GPT4=
     AZURE_OPENAI_DEPLOYMENT_MINI=
     NEWSAPI_KEY=
     DATABASE_URL=postgresql://postgres:secret@localhost:5432/btc
     REDIS_URL=redis://localhost:6379/0
     SESSION_DIR=./data/sessions
     ```
   - `.gitignore` entries: `.env`, `data/`, `__pycache__/`, `.next/`

6. **CI skeleton** (`/.github/workflows/ci.yml`)
   - On push to `dev` / PR to `main`: install deps → lint → run unit tests

### Deliverable
`docker compose up -d` brings up Postgres + Redis; `uv run uvicorn app.main:app --reload` returns `{"status": "ok"}` at `GET /health`.

---

## Phase 1 — Backend Foundation

**Goal:** Stable API skeleton, DB schema, and shared clients (OpenAI, Redis, Postgres).

### Steps

1. **Core config** (`app/core/config.py`)
   - `pydantic-settings` `Settings` class — reads from `.env`
   - Expose: `OPENAI_*`, `DATABASE_URL`, `REDIS_URL`, `SESSION_DIR`

2. **Database setup** (`app/core/db.py`)
   - SQLAlchemy async engine + `AsyncSession` factory
   - Alembic migration folder (`alembic/`)

3. **Database schema** (models in `app/models/`)

   | Table | Key Columns |
   |---|---|
   | `topics` | `id`, `title`, `geography`, `domain`, `heat_score`, `brief_json`, `sentiment_json`, `fetched_at` |
   | `sessions` | `id`, `scenario_type`, `topic_id`, `persona`, `user_segment`, `started_at`, `ended_at`, `transcript_path` |
   | `session_turns` | `id`, `session_id`, `turn_index`, `role` (user/ai), `content`, `tagged_as` (clip/suspicious/null) |
   | `feedback` | `id`, `session_id`, `metrics_json`, `suggestions_json`, `annotated_transcript_path`, `created_at` |
   | `progress` | `id`, `session_id`, `scenario_key`, `attempt_number`, `metrics_json`, `created_at` |

4. **Redis client** (`app/core/redis.py`)
   - `aioredis` async client; helper `get_cached` / `set_cached` with TTL

5. **OpenAI client** (`app/core/openai_client.py`)
   - Thin wrapper around `openai.AsyncAzureOpenAI`
   - Helper `chat_complete(messages, model="gpt-4", temperature=0.7) -> str`
   - Respects `AZURE_OPENAI_DEPLOYMENT_*` env keys

6. **API router structure** (`app/api/`)
   ```
   api/
   ├── topics.py        # GET /topics, GET /topics/{id}/brief
   ├── scenarios.py     # GET /scenarios, POST /scenarios/opponent-brief
   ├── sessions.py      # POST /sessions, GET /sessions/{id}
   ├── simulation.py    # POST /sessions/{id}/turn
   ├── feedback.py      # POST /sessions/{id}/feedback
   └── progress.py      # GET /progress/{scenario_key}
   ```

7. **Health endpoint** — `GET /health` returns `{"status": "ok", "db": "ok", "redis": "ok"}`

8. **CORS** — allow `http://localhost:3000` in dev

### Deliverable
All tables created via Alembic; Swagger UI at `/docs` shows all planned endpoints (even if unimplemented, they return `501`).

---

## Phase 2 — Topic Intelligence Pipeline

**Goal:** Automatically discover, cluster, score, and cache current topics; expose them via API.

### Steps

1. **News fetcher** (`app/services/topic_pipeline.py`)
   - `fetch_articles(query, geography, domain)` — calls NewsAPI `everything` endpoint
   - RSS fallback (simple `feedparser` parsing) if NewsAPI quota exhausted
   - Returns list of `{title, url, source, published_at, snippet}`

2. **Topic clustering & labelling** (LLM call)
   - Send 10–20 article snippets to GPT-4-mini
   - Prompt: cluster into 3–8 distinct topics; return `{topic_title, summary, key_actors, domain, geography}`

3. **Sentiment extraction** (LLM call)
   - Per topic: pass snippets → extract `{overall_sentiment, themes: [...], representative_quotes: [...]}`

4. **Heat score computation**
   ```
   heat_score = (recency_weight * 0.4) + (volume_weight * 0.3) + (sentiment_volatility * 0.3)
   ```
   - `recency_weight`: 1.0 if < 6h, 0.7 if < 24h, 0.3 otherwise
   - `volume_weight`: normalised article count vs. all topics in batch
   - `sentiment_volatility`: std-dev proxy (mix of positive + negative in snippets)

5. **Persistence** — upsert into `topics` table; store full `brief_json` and `sentiment_json`

6. **Redis caching** — after DB write, cache `topic:{id}:brief` and `topic:{id}:sentiment` with TTL of 6h

7. **Scheduled runner** (`scripts/run_topic_pipeline.py`)
   - Simple Python script runnable as a cron: `python scripts/run_topic_pipeline.py`
   - Accepts `--geography` and `--domain` args
   - Calls functions in `topic_pipeline.py` sequentially

8. **API endpoints**
   - `GET /topics?geography=global&domain=economy` — returns list sorted by `heat_score` desc
   - `GET /topics/{id}/brief` — returns cached brief + sentiment; generates if miss
   - `GET /topics/{id}/chat` (POST body: `{question}`) — conversational deep-dive using topic brief as context

### Deliverable
`python scripts/run_topic_pipeline.py` populates topics in Postgres; `GET /topics` returns at least 3 topics with heat scores and briefs.

---

## Phase 3 — Simulation Engine

**Goal:** Turn-based conversation simulations powered by LLM personas; TikTok script generator; journalistic panel simulation.

### Scenario Types (Phase 1)

| ID | Name | Segment |
|---|---|---|
| `tv_debate` | TV Debate / Panel Prep | Political |
| `door_to_door` | Door-to-Door Conversation | Political |
| `tiktok_script` | TikTok Video Script | Political |
| `moderated_panel` | Moderated Televised Discussion | Journalistic |

### Steps

1. **Persona presets** (`app/services/personas.py`)
   - Dict of preset personas with name, description, emotional tendencies, and rhetorical style
   - Political: `young_voter`, `sceptical_urban`, `angry_rural`, `undecided_middle_class`, `key_opinion_leader`
   - Journalistic guests: `expert_pro`, `expert_against`, `evasive_spokesperson`, `angry_citizen`

2. **System prompt builder** (`app/services/prompt_builder.py`)
   - `build_persona_prompt(scenario_type, persona, topic_brief, user_segment)` → system prompt string
   - Includes: role description, topic context, communication style, emotional baseline, constraints (no hate speech, label AI content)
   - For `moderated_panel`: builds prompts for 2 guests simultaneously

3. **Session management** (`app/api/sessions.py`)
   - `POST /sessions` — body: `{scenario_type, topic_id, persona, user_segment}`; creates DB record; returns `session_id`
   - `GET /sessions/{id}` — returns full session metadata + turns so far

4. **Turn endpoint** (`app/api/simulation.py`)
   - `POST /sessions/{id}/turn` — body: `{user_message, tag?}` (tag = `clip_worthy | suspicious | null`)
   - Reads truncated turn history (last N turns to stay within context limit)
   - Calls `openai_client.chat_complete` with system prompt + history
   - Appends both turns to `session_turns`; flushes transcript to `SESSION_DIR/{session_id}.jsonl`
   - Returns `{ai_response, turn_index}`
   - For `moderated_panel`: route `user_message` to specific guest or all; return array of up to 2 AI responses

5. **Opponent briefing** (`app/api/scenarios.py`)
   - `POST /scenarios/opponent-brief` — body: `{public_figure_name, topic_id}`
   - LLM call with web-search simulation (pass topic brief + figure name): returns `{positions_summary, strengths, weaknesses, strategic_hints: [...]}`
   - Note: use only publicly verifiable framing; add disclaimer in response

6. **TikTok script generator** (`app/api/scenarios.py`)
   - `POST /scenarios/tiktok-script` — body: `{topic_id, audience, tone_preference?}`
   - LLM generates 1–3 scripts, each labelled with `{tone: serious|humorous|provocative, risk_level: low|medium|high, script_text}`
   - Optional: `POST /scenarios/tiktok-reactions` — simulate 5–8 audience reactions (supportive, critical, trolling, confused)

7. **Session end** (`POST /sessions/{id}/end`)
   - Marks `ended_at`; finalises transcript JSONL file
   - Returns `{session_id, turn_count, transcript_path}`

### Deliverable
Full conversation turn cycle: create session → send 3+ turns → end session → transcript saved to disk. Verified manually via Swagger UI.

---

## Phase 4 — Feedback & Coaching Module

**Goal:** After each session, analyse the full transcript with GPT-4 and return structured feedback + persisted metrics.

### Steps

1. **Feedback service** (`app/services/feedback_service.py`)
   - `analyse_transcript(session_id)`:
     - Reads transcript from `session_turns` DB table (or JSONL file)
     - Builds feedback prompt based on `scenario_type` and `user_segment`
     - Returns structured JSON (see schema below)

2. **Feedback schema**
   ```json
   {
     "metrics": {
       "resonance": 7,
       "credibility": 6,
       "clarity": 8,
       "respectfulness": 9,
       "on_message": 6
     },
     "highlighted_moments": [
       {"turn_index": 3, "type": "good", "note": "Strong concrete example"},
       {"turn_index": 7, "type": "bad",  "note": "Vague non-answer triggered pushback"}
     ],
     "suggestions": [
       "Lead with the voter benefit before explaining policy mechanics.",
       "Avoid the phrase 'at the end of the day' — low-credibility signal.",
       "When asked about cost, give a number or range immediately."
     ],
     "self_review_prompt": "Looking at turn 7 — what would you say differently now?"
   }
   ```
   - **Political metrics:** resonance, credibility, clarity/structure, respectfulness, on-message score
   - **Journalist metrics:** question_depth, neutral_phrasing, follow_up_quality, verification_behaviour

3. **Annotated transcript** — merge feedback highlights into transcript JSONL; save to `SESSION_DIR/{session_id}_annotated.json`

4. **Persistence** — insert into `feedback` table; insert row into `progress` table with attempt number

5. **API endpoint** — `POST /sessions/{id}/feedback`
   - Triggers analysis if not yet run; idempotent (returns cached if already exists)
   - Returns full feedback JSON

6. **Progress tracking** (`app/api/progress.py`)
   - `GET /progress/{scenario_key}?persona={persona}` — returns list of attempts with metrics for trend visualisation
   - Each item: `{attempt_number, date, metrics, session_id}`

### Deliverable
After ending a session, `POST /sessions/{id}/feedback` returns a full structured feedback object with at least 3 suggestions and 2 highlighted moments.

---

## Phase 5 — Frontend (Next.js)

**Goal:** Clean, functional UI covering the full user journey — topic discovery → scenario setup → simulation → feedback review.

### App Router Pages

| Route | Description |
|---|---|
| `/` | Landing / home — entry point, user segment selector (Political / Journalist) |
| `/topics` | Current Topics view — filterable list, heat score badges |
| `/topics/[id]` | Topic detail — brief, sentiment snapshot, deep-dive chat |
| `/scenarios` | Scenario selection — cards for each scenario type |
| `/scenarios/setup` | Scenario configuration — persona picker, optional opponent brief |
| `/session/[id]` | Active simulation — chat interface, tag buttons (clip / suspicious) |
| `/session/[id]/feedback` | Post-session feedback — metrics, highlights, suggestions, self-review |
| `/progress` | Progress tracker — select scenario + persona, view metric trend lines |

### Steps

1. **Design system** (`src/app/globals.css`)
   - CSS custom properties: colours, typography scale, spacing, border radius, shadows
   - Dark-first colour scheme with teal/gold accent palette
   - Google Font: `Inter` (body) + `DM Serif Display` (headings)

2. **API client** (`src/lib/api.ts`)
   - Typed `fetch` wrapper: `get<T>(path)`, `post<T>(path, body)`
   - Base URL from `NEXT_PUBLIC_API_URL` env var (default: `http://localhost:8000`)
   - Error handling: throw typed `ApiError` with `status` + `message`

3. **Shared components** (`src/components/`)
   - `TopicCard` — title, domain badge, heat score indicator, geography tag
   - `PersonaSelector` — radio-card grid of persona presets
   - `ChatBubble` — user vs. AI message with timestamp + optional tag button
   - `MetricBar` — labelled progress bar (0–10) for feedback metrics
   - `TrendChart` — tiny SVG sparkline for progress tracking (vanilla, no chart lib)
   - `ScenarioCard` — icon, title, description, "Start" CTA

4. **Page implementations** (in order of dependency)
   1. `/topics` — fetch + display topic list; filter controls (geography, domain)
   2. `/topics/[id]` — brief + sentiment; deep-dive chat textarea + response display
   3. `/scenarios` → `/scenarios/setup` — scenario cards; persona + topic form
   4. `/session/[id]` — real-time chat; POST turn on enter; tag buttons; "End Session" button
   5. `/session/[id]/feedback` — render metrics, highlights, suggestions; "Retry" link back to setup
   6. `/progress` — select scenario/persona, plot sparklines per metric across attempts

5. **i18n strings** — keep all UI copy in `src/lib/strings/en.ts`; wired via a simple `t(key)` helper to ease future Slovak localisation

6. **Error & loading states** — every data-fetching page has a loading skeleton and a friendly error fallback

### Deliverable
All 8 routes render without errors; full user flow operable from browser without touching Swagger.

---

## Phase 6 — Integration & End-to-End Slice

**Goal:** Validate two complete vertical slices work end-to-end before declaring MVP done.

### Slice A — Political: TV Debate

1. User lands on `/` → selects **Political**
2. Navigates to `/topics`, picks a topic, reads brief
3. Goes to `/scenarios` → selects **TV Debate** → `/scenarios/setup`
4. Picks `angry_rural` persona, optionally enters opponent name
5. Starts session → `/session/[id]`
6. Exchanges at least 5 turns; tags 1 turn as "clip-worthy"
7. Ends session → `/session/[id]/feedback` → reviews metrics + suggestions
8. Checks `/progress` → sees first attempt charted

### Slice B — Journalistic: Moderated Panel

1. User selects **Journalist** segment
2. Selects topic → `/scenarios/setup` — picks **Moderated Panel**
3. Sees 2 preset guests (conflicting positions)
4. Exchanges 4 turns; asks follow-up to specific guest
5. Tags 1 turn as "suspicious"
6. Ends session → feedback with journalist-specific metrics

### Integration Checklist

- [ ] Backend ↔ Frontend CORS and API routing verified
- [ ] Session turns persisted correctly after page refresh
- [ ] Transcript JSONL file exists on disk after session end
- [ ] Feedback endpoint runs idempotently (safe to call twice)
- [ ] Redis caching: topic brief served from cache on second request
- [ ] Progress table increments attempt count on second run of same scenario+persona
- [ ] Docker Compose teardown and restart — all data persists (Postgres volume)

---

## Phase 7 — Polish & Handoff to Phase 2

**Goal:** Tighten quality, document gaps, and leave the codebase Phase-2-ready.

### Steps

1. **Error handling sweep** — ensure every FastAPI route has try/except; return RFC 7807-style error responses
2. **Non-functional requirements check**

   | NFR | Action |
   |---|---|
   | LLM latency < 3–5 s / turn | Measure average across 10 turns; log to console; switch to `gpt-4o-mini` if over budget |
   | Topic brief generation 30–60 s | Add progress SSE or polling endpoint |
   | Session resilience | Verify partial transcript saved even if `/end` never called |
   | Content safety | Add system-prompt guardrails; label all AI output in UI |
   | Time-to-first-"aha" < 30 min | Walk through Slice A cold; time it |

3. **Seed data script** (`scripts/seed_demo.py`) — 3 pre-generated topics + 1 completed demo session for first pilot/demo without live API calls
4. **Module boundaries** — confirm topic intelligence, simulation engine, and feedback service are importable independently (no circular deps)
5. **README update** — add quickstart commands, link to this plan, note known gaps
6. **Phase 2 backlog stub** — add `BACKLOG.md` with: STT/TTS, Slovak UI, progress dashboards, additional scenarios, Azure deployment Bicep

---

## Tech Decisions Recap

| Concern | Prototype Choice | Future (Phase 2+) |
|---|---|---|
| Frontend | Next.js + Vanilla CSS | Same; WebXR layer added |
| Backend | Python + FastAPI | Same; containerised to Azure Container Apps |
| LLM | Azure OpenAI GPT-4.x / GPT-4.1-mini | Same; swap deployment names |
| DB | PostgreSQL in Docker | Azure Database for PostgreSQL |
| Cache | Redis in Docker | Azure Cache for Redis |
| Auth | None | Azure Entra ID (OAuth2 / OIDC) |
| Speech | Text only | Web Speech API (STT) + Azure AI Speech (TTS) |
| Transcripts | Local filesystem | Azure Blob Storage |
| Topic pipeline | Local cron script | Azure Functions (timer-triggered) |
| Secrets | `.env` | Azure Key Vault |

---

## Out of Scope (Phase 1)

- VR / WebXR / 3D avatars
- Real STT / TTS
- Azure cloud deployment (everything runs locally in Docker)
- Enterprise integrations (LMS, SSO)
- Slovak UI (strings are externalised; translation is a later task)
- Full multilingual beyond SK + EN prompt support
- Large scenario libraries (> 4 scenarios)
- Full privacy/compliance framework

---

*This plan is a living document. Update it as decisions change during implementation.*
