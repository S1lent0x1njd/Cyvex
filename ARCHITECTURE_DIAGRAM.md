# Cyvex — Full-Stack Architecture

Mermaid source below renders on GitHub, VS Code (Markdown Preview
Mermaid extension), and `mermaid.live`. Component responsibilities,
request paths, and degradation behavior follow the diagram.

```mermaid
flowchart TD

    subgraph CLIENTS["Clients"]
        DEV["Developer browser<br/>web console :3000"]
        IDE["VS Code extension<br/>command cyvex.scanFile"]
        CI["CI / CD pipeline<br/>repo scanning job"]
    end

    subgraph FE["Next.js 14 web console (port 3000)"]
        PAGES["App Router pages<br/>dashboard · scanner · alerts · keys<br/>billing · reports · settings · admin"]
        BFF["Route handlers (BFF proxy)<br/>/api/scanner/analyze · /api/scanner/repository<br/>/api/reports/nca-compliance · /api/keys<br/>/api/alerts · /api/users · /api/billing"]
        AUTH["Auth.js (NextAuth) session<br/>+ cyvex_api_jwt HTTP-only cookie<br/>session bridge re-mints JWT on 401"]
    end

    subgraph API["FastAPI backend — uvicorn, port 8000"]
        R_AUTH["auth router<br/>POST /auth/register · POST /auth/login · GET /auth/me"]
        R_SCAN["scanner router<br/>POST /analyze_code (JWT or X-API-Key)<br/>POST /analyze_repository (JWT or X-API-Key, ZIP)"]
        R_ALERT["alert router<br/>GET /alerts · POST /alert/check (X-API-Key)"]
        R_RPT["reports router<br/>GET /reports/nca-compliance (JWT + admin)"]
        R_PLAT["platform routers<br/>keys · users · org · billing (Stripe) · admin"]
        ENGINE["In-process engine stack<br/>ASTParser (tree-sitter, 4 DoS guards)<br/>FeatureEngineer (67 security features)<br/>CyvexPredictor (Random Forest + isotonic)<br/>EvidenceGatherer (15 rules)<br/>ConfidenceScorer (verdict + abstention)<br/>ExplainabilityEngine (ablation drivers)"]
        COPILOT["AIExplainer — google-genai<br/>one plain-English paragraph, fail-closed"]
        PROBLEM["Error/problem layer<br/>{error, message} JSON, 400/401/403/503"]
    end

    subgraph DATA["Persistent state"]
        PG[("PostgreSQL 16<br/>users · orgs · api_keys (bcrypt)<br/>alerts · audit trail")]
        MODEL[("models/cyvex_model.pkl<br/>67 features, calibrated")]
        RULES[("config/verification_rules.yaml<br/>15 evidence rules")]
        GRAPH[("data/deps/graph.json<br/>blast-radius lookups")]
        BENCH[("data/benchmarks/<br/>temporal-CV reports")]
    end

    subgraph WORKERS["Async workers"]
        REDIS[("Redis broker<br/>compose-internal only")]
        CEL["Celery worker + beat<br/>continuous scrape → AST → ML → verify<br/>→ persist alert → Slack Block Kit webhook"]
    end

    subgraph EXT["External services"]
        GH["GitHub REST API<br/>resumable commit scraper"]
        OSV["OSV · NVD · KEV<br/>vulnerability baseline (subtracted, not trusted)"]
        GEM["Gemini API<br/>GOOGLE_API_KEY optional"]
        SLACK["Slack"]
        STRIPE["Stripe<br/>checkout + webhook"]
    end

    subgraph OPS["Secondary surface"]
        GUI["Streamlit GUI — port 8501<br/>dashboard · Pyvis dependency graph · evidence panels"]
    end

    %% web console path
    DEV --> PAGES
    PAGES --> BFF
    AUTH --> R_AUTH
    BFF -->|"cyvex_api_jwt"| R_SCAN
    BFF --> R_RPT
    BFF --> R_PLAT
    BFF --> R_ALERT

    %% IDE path (direct to API, machine plane)
    IDE -->|"X-API-Key: sk_live_..."| R_SCAN

    %% CI path
    CI -->|"X-API-Key"| R_ALERT

    %% engine
    R_SCAN --> ENGINE
    R_ALERT --> ENGINE
    R_RPT --> PG
    R_PLAT --> PG
    R_SCAN --> PROBLEM
    MODEL -.-> ENGINE
    RULES -.-> ENGINE
    GRAPH -.-> ENGINE
    ENGINE --> COPILOT
    COPILOT --> GEM
    ENGINE --> PG

    %% worker loop
    GH --> CEL
    OSV --> CEL
    REDIS --> CEL
    CEL --> ENGINE
    CEL --> PG
    CEL --> SLACK

    %% ops
    GUI --> PG
    STRIPE --> R_PLAT
```

---

## Component responsibilities

| Component | Responsibility | Notes |
|---|---|---|
| **Next.js console** (`frontend/`) | Auth.js login (credentials + Google/GitHub OAuth), dashboard pages, BFF proxies that attach the JWT cookie | port 3000, `FASTAPI_URL` points at the API |
| **VS Code extension** (`vscode-extension/`) | Scans the active editor buffer via `POST /analyze_code` with `X-API-Key`; risk + verdict in a notification, full report in a static webview | zero runtime deps, talks **directly** to FastAPI (no BFF) |
| **FastAPI backend** (`src/api/`) | Auth (JWT sessions + bcrypt API keys), scanner routes, alerts, team/org admin, NCA report, Stripe billing | fail-closed: missing secrets refuse to start |
| **Engine stack** (`src/extractor/`, `src/model/`, `src/verification/`) | parse → features → risk → evidence → verdict → explain | runs **in-process** inside the API request; never raises to the client |
| **PostgreSQL** | Tenancy, credentials, alerts, audit | compose-internal, no host port |
| **Celery worker + Redis** (`src/worker/`) | Continuous repository watching on a beat schedule; persists alerts; Slack delivery with exponential backoff | broker never leaves the compose network |
| **Streamlit GUI** (`gui/`) | Research/ops view: live API probes, Pyvis dependency graph, evidence panels | port 8501 |
| **Gemini** | AI Copilot paragraph only | optional — unset key ⇒ fallback string, scan still succeeds |
| **model bundle / rules / graph** | Versioned artifacts loaded at startup | `run_pipeline.py --check` validates all three |

---

## Request paths

| Path | Auth | Flow |
|---|---|---|
| **Web scan (paste)** | JWT cookie | browser → `/api/scanner/analyze` (BFF) → `POST /analyze_code` → engine → report JSON → 3-column UI + AI paragraph |
| **Web scan (repository)** | JWT cookie | browser → `/api/scanner/repository` (multipart, 180s) → ZIP bounds → per-file engine → aggregated findings |
| **IDE scan** | `X-API-Key` | VS Code → `POST /analyze_code` directly → notification (`risk — verdict`) → webview report |
| **CI / alert check** | `X-API-Key` | pipeline → `POST /alert/check` → verdict + optional Slack webhook |
| **Compliance report** | JWT + admin | console → `/api/reports/nca-compliance` → `GET /reports/nca-compliance` → reportlab PDF |
| **Continuous watch** | internal | Celery beat → GitHub scraper → engine → `alerts` row → Slack |

## Auth planes

| Plane | Credential | Used by | Revocation |
|---|---|---|---|
| Human | `Authorization: Bearer <jwt>` (HS256, bounded TTL) | web console, report generation | secret rotation / TTL expiry |
| Machine | `X-API-Key: sk_live_...` (bcrypt-hashed, prefix-indexed) | VS Code extension, CI | `is_active=false`, per-key |
| Signing | `CYVEX_SIGNING_SECRET` (HMAC-SHA256) | outbound envelopes, fail-closed startup | env rotation |

`/analyze_code` accepts **either** plane (Phase 17): bearer wins when
present, key otherwise, neither ⇒ `401 missing_token` naming both.

## Degradation behavior (never fails a scan silently)

| Failure | Response |
|---|---|
| Model not loaded / predictor throws | `ml.model_available=false`, risk = `NEUTRAL_RISK` (0.5), error code named in JSON |
| Gemini error / no API key | `ai_explanation` = fallback string, scan proceeds |
| Explainability failure | `explain.available=false` with reason, scan proceeds |
| Parser refusal (size/depth/time) | `400 invalid_code` naming the guard |
| Unsupported language | `unsupported_language:<name>` refusal, no crash |
| DB outage on key auth | `503 database_unavailable` |
| JWT invalid + valid key attached | `401 invalid_token` — no silent fallback |

## Fixed pipeline order (never inferred from flags)

```
Data Collection → Feature Extraction → Model Training → Graph Building → Services
  (scrape, ingest)      (extract)           (train)           (graph)     (api, gui)
```

Ports: **3000** Next.js console · **8000** FastAPI · **8501** Streamlit ·
Postgres/Redis compose-internal only.
