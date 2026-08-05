# CineOps Studio
### The Operating System for Film Production

Built on **Google Cloud + Gemini**, with runtime observability powered by **Grafana Cloud**
(Grafana partner track).

## Problem

Film productions lose time and money because scripts, schedules, budgets, and coordination
are fragmented across spreadsheets, group chats, and disconnected tools — location conflicts
get caught on set instead of before it, and budget overruns surface after they're too late to act on.

## Solution

CineOps Studio is a multi-agent production-intelligence platform: upload a screenplay and four
collaborating AI agents — **ScriptBreakdown**, **ScheduleOrchestrator**, **BudgetGuardian**, and
an executive **Producer Copilot** — turn it into a scene breakdown, a conflict-checked shooting
schedule, and a department-level budget forecast, with every agent decision observable live in a
Grafana Cloud control room.

## Demo (3 minutes)

1. Upload a screenplay (or one click loads a pre-cached demo production)
2. ScriptBreakdown extracts scenes, characters, locations, props, VFX shots
3. ScheduleOrchestrator builds a shooting schedule and flags a real location conflict
4. BudgetGuardian forecasts department costs, checking live Grafana metrics as it does
5. Grafana Cloud shows the agents' token burn, latency, and errors in real time
6. Producer Copilot coordinates a cross-cutting change and asks for your approval

Full walkthrough with exact screens and endpoints further down.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     REACT FRONTEND (Vite + TS)                   │
│   Command Center · Script Studio · Schedule Board · Budget Watch │
│                    · Observability (Grafana embed)                │
└───────────────────────────────┬───────────────────────────────────┘
                                 │ REST + WebSocket
┌───────────────────────────────▼───────────────────────────────────┐
│                  FASTAPI BACKEND  (agents built on Google ADK)     │
│  ┌───────────────┐ ┌──────────────────┐ ┌─────────────────────┐  │
│  │ScriptBreakdown│ │ScheduleOrchestrator│ │   BudgetGuardian    │  │
│  └───────┬───────┘ └─────────┬──────────┘ └──────────┬──────────┘  │
│          └───────────────────┼───────────────────────┘             │
│                    ┌──────────────────┐                            │
│                    │ Producer Copilot │  (orchestrates the above,  │
│                    └──────────────────┘   queries Grafana MCP)     │
│                              │                                     │
│              Shared ProductionContext (Firestore)                  │
└──────────────┬───────────────┬───────────────┬─────────────────────┘
               │                │                │
      ┌────────▼──────┐ ┌───────▼──────┐ ┌───────▼────────┐
      │  Gemini 2.5    │ │  BigQuery    │ │ Cloud Storage   │
      │  Flash (Vertex)│ │ (structured) │ │ (PDF uploads)   │
      └────────────────┘ └──────────────┘ └─────────────────┘
               │
      ┌────────▼─────────────────────────────────────┐
      │  OpenTelemetry → Grafana Cloud (OTLP)          │
      │  Traces · Metrics · Grafana MCP (runtime query) │
      └─────────────────────────────────────────────────┘
```

---

## Agents

Every agent reads from and writes to a shared `ProductionContext` (Firestore-backed, in-memory
fallback locally) — none of them operate on isolated state.

| Agent | Purpose | Inputs | Outputs | Tools |
|---|---|---|---|---|
| **ScriptBreakdown** | Extracts structured production data from a screenplay | PDF upload | `ScriptBreakdown` (scenes, characters, locations, props, VFX shots) | Gemini 2.5 Flash (multimodal PDF), Cloud Storage |
| **ScheduleOrchestrator** | Builds a shooting schedule that minimizes company moves and packs night shoots | `ScriptBreakdown` from shared context | `Schedule` with day-by-day call sheets and flagged conflicts | Location-grouping/night-shoot-day algorithm, Gemini (when configured) |
| **BudgetGuardian** | Estimates department costs and flags overrun risk | `ScriptBreakdown` + `Schedule` from shared context | `Budget` with 7 department forecasts, alerts, and a `grafanaNote` | Cost-estimation model, **Grafana MCP** (token burn/latency/error check factored into the forecast) |
| **Producer Copilot** | Executive orchestrator for cross-cutting requests | Natural-language message + shared context | `CopilotResponse` (`reasoning`, `confidence`, `recommendedAction`, `requiresApproval`) | Delegates to the 3 agents above, queries **Grafana MCP** directly |

Each agent's core logic (`extract_script_breakdown`, `build_shooting_schedule`, `analyze_budget`,
`coordinate_production`) is a fully working, independently-callable Python function — that's
what the API routes actually call. Each file also declares a `google-adk` `Agent(...)` wrapper
around that function, imported behind a `try/except ImportError` so the app runs with or without
the package installed. **Verified during testing:** `google-adk` (2.6.2) is a real package and the
`Agent(...)` construction genuinely succeeds against it — but installing it in this project's venv
pulls in a newer `opentelemetry-sdk` that conflicts with the exact version
`opentelemetry-instrumentation-fastapi==0.46b0` requires, breaking FastAPI route registration.
It's intentionally left out of `requirements.txt` until that upstream version conflict resolves;
the app is fully functional without it since the routes call the underlying functions directly.

---

## Observability & Grafana

**Why Grafana:** agent decisions here aren't a black box — BudgetGuardian and Producer Copilot
actually change their output based on live system metrics, and that has to be visible for the
integration to mean anything.

**How OpenTelemetry connects:** the backend's `TracerProvider`/`MeterProvider`
(`app/services/telemetry.py`) export traces and metrics via OTLP to Grafana Cloud when
`GRAFANA_OTLP_ENDPOINT`/`GRAFANA_OTLP_USERNAME`/`GRAFANA_API_KEY` are set (Basic auth per
Grafana Cloud's OTLP gateway convention), falling back to console export locally. Every agent
invocation emits `agent.tokens_used`, `agent.latency`, `agent.mcp_calls`, `agent.errors`, plus
production-level gauges (`production.scenes_extracted`, `production.conflicts_open`,
`budget.burn_rate`). FastAPI itself is auto-instrumented for request/latency spans.

**How Grafana MCP is used at runtime:** `app/services/grafana_mcp.py`'s `GrafanaMCPClient`
queries Grafana Cloud's MCP server for token burn, p95 latency, and error count. BudgetGuardian
calls this *during* every budget analysis and bumps the VFX forecast if token burn is
accelerating — the result is surfaced directly in the UI as a `grafanaNote`, not just logged.
Producer Copilot queries it independently too, factoring system health into its confidence score.
If Grafana is unreachable, both agents fall back gracefully (never a hard failure) — with a
clearly-labeled simulated snapshot in local/demo mode rather than silently returning nothing.

**How the dashboard is accessed:** `grafana/dashboards/cineops-dashboard.json` (UID
`cineops-production`) has 7 panels — token burn rate, p95 latency, MCP call count, agent errors
(red-thresholded), scenes extracted, open conflicts, and an AI traces panel — importable directly
into a Grafana Cloud stack. The frontend's Observability page embeds it live via `VITE_GRAFANA_URL`,
with an automatic Recharts fallback (fed by `/metrics/summary`) if the iframe ever fails to load.

### Partner track — how this satisfies the Grafana requirements

- Real OTLP trace/metric export from the backend (not just a static dashboard screenshot)
- Grafana Cloud dashboard JSON, ready to import, with panels tied to this project's actual metric names
- **Runtime** Grafana MCP integration — two different agents query it mid-decision, and the result changes their output
- A resilience story for the observability layer itself (kill-switch fallback), not just the happy path

---

## Repo structure

```
cineops-studio/
├── frontend/    React 18 + TypeScript + Vite + Tailwind
├── backend/     FastAPI + agents + services
├── grafana/     dashboards/cineops-dashboard.json
├── demo/        pre-cached JSON + lunar_echo.pdf (demo screenplay) + storyboards/ (3 PNG frames)
├── deploy/      cloudbuild.yaml, service.yaml
└── pitch-deck/  5-slide pitch deck (.pptx)
```

## Local setup

**Backend**
```bash
cd backend
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # leave PROJECT_ID empty to run fully in mock mode
uvicorn app.main:app --reload --port 8000
# API docs: http://localhost:8000/docs
pytest                       # 32 tests, all pass against the mock/local path
```

**Frontend**
```bash
cd frontend
npm install
cp .env.example .env         # leave all vars empty to run against mock data
npm run dev                  # http://localhost:5173
```

Both sides run standalone with zero configuration — every GCP/Grafana-dependent service
(Gemini, BigQuery, Firestore, Cloud Storage, Grafana MCP) has a local fallback, so you can
develop and demo the full flow without any cloud credentials.

## Deployment

```bash
gcloud builds submit --config=deploy/cloudbuild.yaml \
  --substitutions=_REGION=us-central1
```
`deploy/cloudbuild.yaml` builds and deploys both services to Cloud Run.
`deploy/service.yaml` is the Knative service spec for the backend (2 CPU / 2Gi, per the
build checklist) if you prefer `gcloud run services replace` over Cloud Build.

> **Honest limitation:** this was built and tested in an environment with no network access
> to `googleapis.com` or `grafana.com` and no GCP credentials. Every real-cloud code path
> (Gemini calls, BigQuery, Firestore, Cloud Storage, Grafana OTLP/MCP) is implemented and
> gated behind `PROJECT_ID`/`GRAFANA_*` config, but has not been exercised against live
> services — only the local-fallback path has been. Actual deployment, DNS/SSL, and a live
> Grafana Cloud dashboard import are next steps for whoever has the credentials.

## Environment variables

| Variable | Where | Purpose |
|---|---|---|
| `PROJECT_ID` | backend | GCP project — empty runs backend in mock mode |
| `REGION`, `VERTEX_LOCATION` | backend | GCP region (default `us-central1`) |
| `GEMINI_MODEL` | backend | `gemini-2.5-flash-preview-05-20` |
| `BQ_DATASET`, `BQ_LOCATION` | backend | BigQuery dataset (`cineops`, `US`) |
| `GCS_BUCKET` | backend | Cloud Storage bucket for PDF uploads |
| `FIRESTORE_COLLECTION` | backend | Shared production context collection |
| `GRAFANA_CLOUD_URL`, `GRAFANA_API_KEY` | backend | Grafana MCP client auth |
| `GRAFANA_OTLP_ENDPOINT`, `GRAFANA_OTLP_USERNAME` | backend | OTLP trace/metric export |
| `BACKEND_URL`, `FRONTEND_URL` | backend | Self-referential URLs, used for CORS/config wiring at deploy time |
| `CORS_ORIGINS` | backend | Comma-separated allowed origins (never `*`) |
| `VITE_API_URL` | frontend | Backend base URL — empty runs frontend on mock data |
| `VITE_WS_URL` | frontend | WebSocket base URL for chat |
| `VITE_GRAFANA_URL` | frontend | Grafana dashboard URL for the Observability embed |

## API endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/scripts/upload` | Upload a screenplay PDF → ScriptBreakdown |
| GET | `/scripts/{production_id}` | Fetch a saved breakdown |
| POST | `/schedules/{production_id}/generate` | Run ScheduleOrchestrator |
| GET | `/schedules/{production_id}` | Fetch a saved schedule |
| POST | `/budgets/{production_id}/analyze?total_budget=` | Run BudgetGuardian (includes a live Grafana MCP check, surfaced as `grafanaNote`) |
| GET | `/budgets/{production_id}` | Fetch a saved budget |
| POST | `/copilot/chat` | Producer Copilot — REST alternative to the WebSocket chat, same `CopilotResponse` shape |
| WS | `/chat/{production_id}` | Multi-agent chat, `@mention` or auto-routed, standardized `{event, data}` envelope (see below) |
| GET | `/agents` | Agent registry — name, description, live status |
| GET | `/agents/status` | Just the status map, keyed by agent name |
| GET | `/timeline/{production_id}` | 8-stage pipeline, derived from actual production state |
| GET | `/risk/{production_id}` | Heuristic risk score (0–100), level, contributing factors |
| GET | `/continuity/{production_id}` | Continuity warnings from the adjacent-scene heuristic |
| POST | `/continuity/check` | Re-run the continuity check on demand |
| POST | `/storyboard` | Generate/fetch the storyboard frames for a production |
| GET | `/storyboard/{id}` | Fetch a single storyboard frame |
| GET | `/storyboards/{filename}` | Static file serving for the storyboard PNGs |
| GET | `/health/`, `/health` | Liveness check (both forms accepted) |
| GET | `/ready` | Readiness — reports whether GCP/Grafana are configured |
| GET | `/live` | Bare liveness probe |
| GET | `/version` | App version + configured Gemini model |
| GET | `/metrics/summary` | Recharts fallback data for the Observability kill-switch |

**Standardized WebSocket events** (`{event, data}` envelope, sent over `/chat/{production_id}`):
`script.started`, `script.completed`, `schedule.generated`, `budget.updated`, `agent.typing`, `agent.finished`, `production.updated`, `alert.created`. Emitted consistently whether triggered via chat or the equivalent REST endpoints, so any connected client stays in sync.

Full interactive docs at `/docs` once the backend is running (FastAPI's auto-generated Swagger UI).

## Demo walkthrough (3 minutes, in detail)

1. **Hook** — "Film productions lose $50B annually to coordination failures."
2. **Demo Accelerator** — one click in the top bar loads "Lunar Echo" instantly (breakdown,
   schedule, budget) and pre-populates chat with a sample Producer Copilot exchange, including
   a live Decision Panel — no live typing required if the network is unreliable.
3. **AI Production War Room** (Command Center) — click "Run Production" and watch Producer
   Copilot → ScriptBreakdown → ScheduleOrchestrator → BudgetGuardian → Grafana MCP → Final
   Recommendation animate through in sequence. This isn't a fake animation — it's actually
   calling `/schedules/{id}/generate` and `/budgets/{id}/analyze` live.
4. **Schedule Board** — Day 3 shows a real `LOCATION_DOUBLE_BOOKED` conflict, computed by
   ScheduleOrchestrator's actual location-grouping/night-shoot-day algorithm.
5. **Budget Watch** — VFX flagged 15%+ over allocation, with both a `reasoning` field and a
   visible "Grafana MCP" note showing the live token-burn/latency/error check that fed the forecast.
6. **Observability** — Grafana Cloud control room; if the live iframe ever fails mid-demo,
   the Agent Telemetry tab is a full Recharts fallback fed by the same `/metrics/summary` data.
7. **Chat** — ask the Producer Copilot "we lost our lead actor on Day 2" and watch it coordinate
   ScheduleOrchestrator + BudgetGuardian + Grafana MCP into one synthesized answer with a
   confidence score and an Approve/Reject decision panel.

## Tech stack

React 18 · TypeScript · Vite · Tailwind · TanStack Query · Zustand · FastAPI · Pydantic ·
Google ADK (agent framework) · Gemini 2.5 Flash (Vertex AI) · BigQuery · Firestore ·
Cloud Storage · Cloud Run · OpenTelemetry · Grafana Cloud (partner track)

## Roadmap

- **LocationScout** — location-scouting agent (permits, weather risk, travel time)
- **CrewMatch** — crew/vendor matching agent
- **Voice Producer** — full Gemini Live bidirectional audio (currently a gated stub in the UI)
- Long-term production memory (Vertex AI memory across sessions)
- Full carbon/sustainability dashboard (currently a single heuristic card)

## License

MIT — see [LICENSE](./LICENSE).

## Credits

Built with Google Cloud (Vertex AI, BigQuery, Firestore, Cloud Storage, Cloud Run), Gemini 2.5
Flash, Google ADK, and Grafana Cloud (dashboards, OTLP ingestion, MCP) with OpenTelemetry for
instrumentation.

## 👥 Team

### Muazu Abdullahi Muhammed
**Founder • AI Engineer • Full-Stack Developer • Product Designer**

**Responsibilities**
- Conceived and led the CineOps Studio product vision.
- Designed the overall system architecture and multi-agent workflow.
- Developed the FastAPI backend and Google ADK agent orchestration.
- Built and integrated the Producer Copilot, ScriptBreakdown, ScheduleOrchestrator, and BudgetGuardian agents.
- Integrated Google Gemini, Vertex AI, Firestore, BigQuery, Cloud Storage, and Grafana Cloud.
- Designed and implemented the React frontend, dashboards, and production workflow UI.
- Implemented OpenTelemetry instrumentation and observability.
- Developed the Demo Accelerator, production timeline, risk analysis, and sustainability features.
- Managed deployment to Google Cloud Run and project infrastructure.
- Prepared the technical documentation, presentation, and hackathon submission.
