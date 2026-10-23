Review
# CareerPilot AI

Private, single-user agentic job-search copilot: parse a resume, score a job description, run a mock interview, and track applications.

## Architecture

See `docs/ARCHITECTURE.md`. Edge TLS in AWS is **nginx on EC2** (self-signed), not an ALB.

```
Browser --HTTPS allowlisted IP--> nginx (EC2)
                                 |-- frontend (React)
                                 |-- backend (FastAPI + LangGraph)
                                      |-- Postgres + pgvector
                                      |-- Claude via app/llm/client.py
                                      |-- MiniLM embeddings (local)

```

## Local setup

```bash
cp .env.example .env
# set JWT_SECRET and SEED_USER_PASSWORD; leave LLM_MODE=fake for tests
docker compose up --build

```
- API: http://localhost:8000/healthz
- UI: http://localhost:5173 (compose frontend) or http://localhost (nginx)
- Login: `SEED_USER_EMAIL` / `SEED_USER_PASSWORD` from `.env`

Backend only:

```bash
cd backend
python -m venv .venv
.venv/Scripts/activate   # Windows
pip install -e ".[dev]"
python tests/generate_fixtures.py
alembic upgrade head
python -m app.db.seed
LLM_MODE=fake uvicorn app.main:app --reload

```

Frontend only: `cd frontend && npm install && npm run dev`

## Tests

Tests **never** call Anthropic (`LLM_MODE=fake` / `FakeLLM`).

```bash
cd backend && pytest --cov=app/agents --cov=app/tools --cov-fail-under=70
cd frontend && npm test && npm run lint
cd frontend && npx playwright test   # needs compose stack

    Client -->|HTTPS / SSE| Gateway
    Gateway --> Router
    Router --> Harness
    Harness --> Orch
    Orch --> RA & JA & IA & TA & CC
    RA & JA & IA --> Guardrails
    TA --> CommitCoord
    CommitCoord --> RelDB
    RA & JA --> VecDB & FTS
    Harness --> Claude
```
## 🤖 The Agent Fleet
The system operates **8 specialized agents** working in concert:
1. **Orchestrator Agent (Router)**: Evaluates user intent using structured LangGraph state routing and can chain sub-agents (e.g., triggering Job-Fit immediately after resume ingestion).
2. **Resume Intelligence Agent**: Extracts candidate entities, decomposes work history into atomic bullet chunks, embeds vectors, and populates searchable lexical profiles.
3. **Job-Fit Analyst Agent**: Computes keyword coverage, match deltas, and synthesizes tailored bullet points constrained by retrieved source chunks.
4. **Interview Coach Agent**: ReAct-style probing loop generating targeted technical/behavioral questions, scoring answers against STAR rubrics, and challenging the candidate's weakest dimension.
5. **Application Tracker Agent**: Tool-calling agent converting conversational instructions into typed, validated, and staged database mutations.
6. **Chitchat Agent**: Handles general guidance, career navigation, and casual interactions via guarded live streams.
7. **Memory Writer Agent**: Extracts durable profile facts (`target_role`, `location_pref`, `salary_pref`) with provenance caps ($\le 0.6$ for untrusted resume/JD sources) and temporal supersession.
8. **Session Summarizer Agent**: Compresses historical interview transcripts into STAR averages and compact working-memory context windows.
---
  Recommended Action: Review agent prompts and test case
----------------------------------------------------------------------
SPAN TREE (4 spans):
  [OK]    retrieval  | hybrid_search             |     42ms | Tokens: 0+0
  [OK]    llm        | step_1                    |   1120ms | Tokens: 1240+310
  [OK]    gate       | final_gate                |      3ms | Tokens: 0+0
  [OK]    db         | commit_coordinator        |     12ms | Tokens: 0+0
======================================================================
```
---
## 🚀 Deployment
### Option A: 1-Click Deploy via Render Blueprint
1. Fork or push this repository to GitHub.
2. Log into [Render](https://render.com/).
3. Create a new **Blueprint** and point to your repository.
4. Render detects [`render.yaml`](render.yaml) and provisions PostgreSQL (with `pgvector`), FastAPI backend, and React static frontend automatically.
### Option B: AWS Production via Terraform
The `infra/` folder provisions an isolated, SG-restricted deployment:
- **Port 443 only** exposed to your allowlisted IP CIDR.
- **Port 22 closed**: Instance access via AWS Systems Manager (SSM) Session Manager.
- **Secrets**: Auto-hydrated from AWS Secrets Manager.
- **State Storage**: Remote S3 state backend with DynamoDB locking.
```bash
cd infra/terraform
terraform init
terraform apply -var-file=envs/dev.tfvars
```
---
## 📂 Repository Structure
```
├── .claude/                    # Tool hook configurations for development
├── backend/
│   ├── alembic/                # Database migrations (0001_init, 0002_harness)
│   ├── app/
│   │   ├── agents/             # Sub-agents (orchestrator, resume, jobfit, interview, tracker)
│   │   ├── api/                # REST endpoints (auth, chat, jobs, resumes, applications, memory, runs)
│   │   ├── db/                 # SQLAlchemy models, session, seed scripts
│   │   ├── harness/            # Safety runtime: hooks, loop, gates, memory, tools, tracing, retrieval
│   │   │   ├── memory/         # Procedural, semantic, episodic, writer, summarizer
│   │   │   ├── retrieval/      # Hybrid RRF, semantic pgvector, lexical tsvector
│   │   │   ├── tools/          # ToolSpec, ToolRegistry, ToolExecutor
│   │   │   ├── tracing/        # PostgresTracer, LangfuseTracer, redaction, cost
│   │   │   └── procedural_assets/ # Versioned prompt MDs, STAR rubrics, skills
│   │   └── llm/                # Claude client, FakeLLM, prompt definitions, guardrails
│   └── evals/                  # Golden test datasets, metrics, judge, cassette runner
├── frontend/
│   ├── src/
│   │   ├── api/                # Typed API client, SSE streaming reader
│   │   ├── components/         # Chat window, Kanban board, Analytics, UI library
│   │   └── pages/              # Dashboard, Login, Interview Coach
├── infra/                      # Terraform modules (network, compute, security, storage)
├── scripts/                    # diagnose_run.py, run_evals.py, promote_run_to_eval.py, hooks
└── docs/                       # HARNESS.md, EVALS.md, DEBUGGING.md, DECISIONS.md
```
---
