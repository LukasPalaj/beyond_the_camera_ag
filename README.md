# Beyond the Camera

> **AI + VR-ready simulation platform for high-stakes communication training**

Beyond the Camera is an AI-driven practice environment where political candidates, spokespeople, and journalists can rapidly get up to speed on current topics, rehearse realistic conversations and media appearances, receive structured feedback, and track their improvement over time.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Vision & Value Proposition](#vision--value-proposition)
- [Target Users](#target-users)
- [Features (MVP)](#features-mvp)
- [Out of Scope (v1)](#out-of-scope-v1)
- [Architecture](#architecture)
- [Prototype Decisions](#prototype-decisions)
- [Local Development](#local-development)
- [Non-Functional Requirements](#non-functional-requirements)

---

## Overview

| Field | Details |
|---|---|
| **Working title** | Beyond the Camera |
| **Type** | AI + VR-ready simulation platform |
| **Initial segments** | Political parties · Journalists / media professionals |
| **First prototypes** | Desktop web app (VR as medium-term extension) |
| **Languages** | Slovak (primary) · English (content via LLMs) |

---

## Problem Statement

### Political Parties
Candidates and spokespeople face high-stakes TV debates, interviews, door-to-door canvassing, and social media. Current preparation is:
- Ad hoc, expensive, and hard to scale
- Weakly connected to real-time voter sentiment

**Result:** Inconsistent messaging, avoidable gaffes, under-used voter data.

### Journalists & Media
Journalists juggle tight deadlines, complex topics, and high expectations of credibility. Training today:
- Focuses on theory or one-off workshops
- Rarely simulates hostile interviews, live panels, or press conferences
- Almost never integrates real-time topic and sentiment intelligence

**Result:** Verification shortcuts, difficulty with nuanced topics, eroding audience trust.

---

## Vision & Value Proposition

> **Become the standard "flight simulator for high-stakes communication"** across politics and journalism, rooted in Murrow's principle: truthfulness → credibility → believability → persuasive impact.

| Segment | Value Proposition |
|---|---|
| **Political parties** | Train candidates on *what* to say and *how* to say it — in scenarios driven by real voter topics and emotions, with unlimited practice and structured feedback. |
| **Journalists** | A preparation copilot that helps you map the information landscape, understand a topic fast, anticipate angles and questions, and rehearse interviews and panels — while reinforcing credibility and verification discipline. |

---

## Target Users

### Segment 1 — Political Party

| Persona | Goals | Success Metric |
|---|---|---|
| **Candidate** | Confidence, message control, voter resonance | More relevant content published; better feedback from team and media |
| **Campaign Manager / Head of Comms** | Message discipline, risk reduction, slate preparedness | Fewer on-air disasters, more consistent messaging |

### Segment 2 — Journalists / Media

| Persona | Goals |
|---|---|
| **TV / Radio host or moderator** | Lead fair, sharp, credible discussions; ask the right follow-ups |
| **Reporter / Correspondent** | Quickly grasp new topics, write credible stories, run solid interviews |
| **Editor / Head of Training** | Scalable, modern training that actually improves on-air / in-print quality |

---

## Features (MVP)

### 🗺️ Current Topics & Sentiment Intelligence
- **Current Topics view** — filterable by geography (global / national / local) and domain (healthcare, defence, economy, environment, education…)
- **Topic Brief** — AI-generated 1–2 paragraph summary: what happened, key actors, why it matters, who it affects
- **Sentiment Snapshot** — overall sentiment, 3–5 dominant themes, 3–10 representative quotes
- **Hot Topics suggestions** — ranked by a heat score (recency + volume + sentiment volatility) per voter group
- **Deep-dive chat** — after selecting a topic, access historical background, root causes, key arguments for/against, source-linked facts, and a chat interface for further exploration

### 🎬 Scenario Selection & Configuration
Available scenarios:

| Type | Scenarios |
|---|---|
| **Political** | TikTok video post · TV debate / panel prep · Door-to-door conversation |
| **Journalistic** | Moderating a televised discussion on a polarising issue |

- Select or manually enter topic
- Choose persona preset for your counterpart (young voter, skeptical urban voter, angry rural voter, undecided middle-class voter, key opinion leader, angry citizen, evasive spokesperson…)
- Optional **opponent assessment**: enter a public figure's name to get a summary of their positions, strengths/weaknesses, and 3–5 strategic hints

### 🤖 Simulation Engine
- **Conversation simulations** (TV debate, door-to-door, moderated panel): turn-based, LLM-driven counterparts with speech input and TTS output; full transcript logged per turn
- **TikTok script simulation**: generate 1–3 script options labelled with tone (serious / humorous / provocative) and risk level; optional simulated audience reactions (supportive, critical, trolling, confused)
- **Journalist moderated panel**: at least 2 guests with conflicting positions; moderator can ask open questions and follow-ups to specific guests

### 📊 Feedback & Improvement Tracking
- Post-session feedback screen: full transcript, highlighted good/bad moments, 3–5 scenario-specific metrics
- **Political metrics:** resonance with target persona, credibility, clarity/structure, respectfulness vs. hostility, staying on message
- **Journalist metrics:** question depth and sharpness, neutral vs. loaded phrasing, follow-up quality, verification behaviour
- 3–5 concrete improvement suggestions per session
- Self-review prompts ("What would you do differently?")
- **Progress tracking:** re-run same scenario and compare metrics across attempts with visual trend lines

### 🎙️ Journalist-Specific Tools
- Pre-simulation topic brief + suggested opening questions and angles
- Ability to tag moments as "clip-worthy" or "suspicious" during simulation
- Feedback explicitly flags missed follow-ups, biased phrasing, and missed verification opportunities

### 🗳️ Political-Specific Tools
- Topic and persona aligned to target electorate segment
- Feedback shows what resonated vs. what triggered backlash or confusion
- Alternative framings suggested ("Instead of saying X… try Y…")

---

## Out of Scope (v1)

- Full multi-user VR environments; advanced 3D avatars and motion capture
- Enterprise integrations (LMS, SSO, HR systems)
- Full privacy/compliance framework for sensitive party/newsroom data
- Large scenario libraries (start with 1–2 deep scenarios per vertical)
- Full multilingual support beyond Slovak + English

---

## Architecture

```
SPA Frontend  →  API / Orchestration (Container Apps)  →  Azure OpenAI + PostgreSQL + Blob + Redis
```

### Azure Components

| Layer | Service | Purpose |
|---|---|---|
| **Frontend** | Azure Static Web Apps / App Service | React / Next.js SPA; WebXR client later |
| **Backend / API** | Azure Container Apps | Stateless REST/GraphQL API (Python/FastAPI or Node/NestJS) |
| **LLMs** | Azure OpenAI (GPT-4.x / GPT-4.1-mini) | Topic briefs, persona dialogue, feedback & coaching |
| **Relational DB** | Azure Database for PostgreSQL Flexible Server | Users, sessions, scenarios, topic catalogue, metrics |
| **Object Storage** | Azure Blob Storage | Transcripts, generated scripts, future audio/video |
| **Cache** | Azure Cache for Redis | Topic briefs, sentiment snapshots, persona prompts (TTL: 6–24 h) |
| **Topic Pipeline** | Azure Functions (timer-triggered) | Pull news/social feeds, cluster topics, compute heat score |
| **Identity** | Microsoft Entra ID | OAuth2 / OIDC; B2C later if needed |
| **Secrets** | Azure Key Vault | API keys, connection strings |
| **Observability** | Application Insights + Log Analytics | Latency, usage, LLM errors |
| **CI/CD** | GitHub Actions + Bicep/Terraform | Build, deploy frontend + backend containers |

### Key Data Flows

1. **Topic discovery (batch)** — Timer Function pulls feeds → LLM clusters & labels topics → LLM extracts sentiment/themes → persisted to PostgreSQL + Redis
2. **Scenario setup** — User selects topic → backend reads brief from Redis/PostgreSQL → optional opponent briefing via LLM → user picks scenario + persona
3. **Conversation simulation** — Session created → system prompt bootstraps persona → turn-based WebSocket loop with truncated history → transcript flushed to Blob/DB
4. **TikTok script** — User selects topic + audience → LLM generates 1–3 labelled scripts → optional LLM-generated reactions
5. **Feedback & coaching** — Transcript + metadata sent to LLM coaching analyst → scores + suggestions stored in PostgreSQL; full annotated JSON in Blob → progress trend queries on re-run

---

## Prototype Decisions

The following decisions apply to the **first prototype phase** only. They reduce complexity so the team can move fast and validate value before wiring up full cloud infrastructure.

### 🖥️ Frontend
- **Framework:** Next.js (React-based, SSR-ready, simplifies future Entra auth integration)
- **Styling:** Vanilla CSS (no Tailwind for now)

### ⚙️ Backend
- **Language & framework:** Python + FastAPI — best fit for LLM/AI tooling ecosystem
- **LLM calls:** Plain `openai` Python SDK (no LangChain/LangFlow in production code; LangFlow used only for offline prompt prototyping)

### 🔐 Authentication
- **Prototype:** No authentication — single shared session, no login screen
- **Phase 2+:** Azure Entra ID (OAuth2 / OIDC)

### 🗣️ Speech (STT / TTS)
- **Prototype:** Text-only input and output (type to speak, read the reply)
- **Phase 2:** Browser Web Speech API for STT + Azure AI Speech or OpenAI TTS for voice output

### 🌐 Language / i18n
- **Prototype UI language:** English (Slovak applied in a later pass once content & flows are stable)
- **LLM content:** English and Slovak both supported via prompts from day one
- All UI strings will be kept in external resource files from the start to make localisation straightforward

### 🗄️ Data & storage (local vs cloud)
| Concern | Prototype (local) | Production (Azure) |
|---|---|---|
| Relational DB | PostgreSQL in Docker | Azure Database for PostgreSQL Flexible Server |
| Cache | Redis in Docker | Azure Cache for Redis |
| Transcripts / artefacts | Local filesystem (`/data/sessions/`) | Azure Blob Storage |
| Topic pipeline | Local cron / scheduled script | Azure Functions (timer-triggered) |
| Secrets | `.env` file (git-ignored) | Azure Key Vault |
| Observability | Console + local log files | Application Insights + Log Analytics |

### 🔑 API keys (prototype)
- Azure OpenAI endpoint, deployment names, and API key stored in a local `.env` file
- News/topic data source: **NewsAPI** (`newsapi.org`) — API key also in `.env`; RSS feeds as fallback

### 📦 Project structure (planned)
```
beyond_the_camera_ag/
├── backend/          # FastAPI app
│   ├── app/
│   │   ├── api/      # Route handlers
│   │   ├── core/     # Config, DB, Redis, OpenAI clients
│   │   ├── models/   # SQLAlchemy / Pydantic models
│   │   ├── services/ # Topic pipeline, simulation, feedback
│   │   └── main.py
│   ├── pyproject.toml
│   └── .env.example
├── frontend/         # Next.js app
│   ├── src/
│   │   ├── app/      # Next.js App Router pages
│   │   ├── components/
│   │   └── lib/      # API client, hooks
│   └── package.json
├── scripts/          # Local topic-pipeline cron, seed data
├── docker-compose.yml
└── README.md
```

---

## Local Development

### Prerequisites

Install via [Homebrew](https://brew.sh):

```bash
# Shell & terminal
brew install iterm2
# Oh My Zsh (separately)

# Python toolchain
brew install pyenv
pip install uv  # or poetry

# Node toolchain
brew install nvm

# Containers & cloud
brew install --cask docker
brew install azure-cli git gh
npm install -g azure-functions-core-tools@4
```

### Local Services (Docker)

```bash
# PostgreSQL
docker run -d --name btc-postgres -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16

# Redis
docker run -d --name btc-redis -p 6379:6379 redis:7

# Langfuse (LLM observability)
# See https://langfuse.com/docs/deployment/local

# LangFlow (visual chain designer)
# pip install langflow && langflow run
```

### Running the Project

```bash
# Backend (FastAPI example)
cd backend
uv sync
uv run uvicorn app.main:app --reload

# Frontend
cd frontend
nvm use
npm install
npm run dev
```

### Recommended IDE Extensions (VS Code / Cursor / Antigravity)

- Python, Jupyter
- Docker
- GitLens
- ESLint + Prettier
- REST Client / Thunder Client
- Azure Account, Azure Resources, Azure Functions

### Testing APIs

Use the **REST Client** or **Thunder Client** extension, or external tools such as Postman, Bruno, or Insomnia. Point requests at `http://localhost:8000` (backend) and `http://localhost:3000` (frontend).

---

## Non-Functional Requirements

| Category | Target |
|---|---|
| **LLM response latency** | < 1 s ideal, < 3–5 s maximum (per turn) |
| **Topic brief generation** | 30–60 s acceptable; show progress; cached per topic |
| **Graceful degradation** | If live topic intelligence fails, allow manual topic entry |
| **Session resilience** | Partial transcripts saved on API failure |
| **Privacy (prototype)** | Public/synthetic data only; minimal personal identifiers; basic access control |
| **Content safety** | No hate speech; no incitement; AI-generated content clearly labelled |
| **Opponent analysis** | Only publicly verifiable info; no defamatory strategies |
| **Time-to-first-"aha"** | < 30 minutes from sign-in to completed simulation + feedback |
| **Modularity** | Topic intelligence, simulation engine, and feedback analysis are independently swappable modules |
| **Observability** | Scenario type, topic, persona, session duration, turn count, feedback metrics all logged |
| **Cost efficiency** | Cache topic briefs (6–24 h TTL); pre-generate canned demos for first pilots |

---

## Roadmap Highlights

- **Phase 1 (now):** Two working vertical slices — 3 political scenarios + 1 journalistic scenario; minimal topic-intelligence pipeline; session feedback loop
- **Phase 2:** Additional scenarios, richer sentiment intelligence, progress dashboards, multi-language support
- **Phase 3:** VR front-end (WebXR / Unity), advanced avatar & body-language analysis, enterprise integrations

---

*Beyond the Camera is an experimental tool. It is intended for training purposes only and should not be used for real confidential strategy. All AI-generated content is clearly labelled as such and does not represent official advice or real individuals.*
