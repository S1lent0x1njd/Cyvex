# Cyvex — Silent-Fix Detection Platform

**Detecting silently-committed security fixes in open source — with an AI
Copilot that explains every verdict, an IDE surface, and the compliance
report your auditor asks for.**

Up to **50% of security fixes in open-source repositories are committed
silently** — no CVE, no advisory, no changelog entry — and more than
**10% have active exploits in the wild** before anyone outside the project
knows (Dong et al., DSN 2025; Li & Paxson, CCS 2017). Advisory-driven
tools (Snyk, Dependabot, OSV) are blind to that half **by construction**:
they read feeds silent fixes never enter. Cyvex watches the commits
themselves, scores them with a calibrated AST model, and refuses to
issue a verdict the evidence cannot support.

| | |
|---|---|
| **Web console** | Next.js 14 + Auth.js — alerts, scanner, keys, billing, team admin |
| **Scanner** | paste-code and ZIP repository scans (Python, JS/TS, Go, Java) |
| **AI Copilot** | Gemini plain-English explanation per verdict, fail-closed |
| **VS Code extension** | scan the active file in-editor with an `X-API-Key` |
| **NCA reports** | one-click ECC-2:2024 compliance PDF from live data |
| **Engine** | tree-sitter AST (4 DoS guards) → 67 features → Random Forest → 15-rule Verification Gate |
| **Workers** | Celery + Redis continuous repo watching → Slack alerts |
| **Licensing** | offline EdDSA (Ed25519) JWT licenses, hardware-bound, fail-closed |

---

## Architecture

```
┌─────────────────────────────┐
│ Clients                     │
│   Next.js console  :3000    │──── JWT cookie ──┐
│   VS Code extension        │──── X-API-Key ───┤
│   CI pipeline              │──── X-API-Key ───┤
│   cyvex CLI (this repo)    │──── X-API-Key ───┤
└─────────────────────────────┘                  │
                                                 ▼
┌──────────────────────────── Cyvex stack (docker compose) ────────────┐
│  api       FastAPI :8000   auth · scanner · alerts · billing ·       │
│                            licensing (Ed25519 keyring, startup assert)│
│  worker    Celery + beat   hourly scrape → ingest → predict → verify │
│  gui       Streamlit :8501 ops dashboard (graph + evidence)          │
│  postgres  PostgreSQL 16   tenants · keys · alerts · licenses        │
│  redis     Redis           broker + result backend (internal only)   │
└──────────────────────────────────────────────────────────────────────┘
        │                      │                        │
        ▼                      ▼                        ▼
  GitHub / OSV / NVD      Gemini API                 Slack
```

Pipeline order is fixed, never inferred from flag order:

```
Data Collection → Feature Extraction → Model Training → Graph Building → Services
  (scrape, ingest)      (extract)           (train)           (graph)     (api, gui)
```

The verdict chain (the "Verification Gate"):

```
features ──► EvidenceGatherer ──► 15 rule hits + benign flags
   │                                    │
   ├──► CyvexPredictor ──► risk score   │
   │                                    ▼
   └─────────────────────► ConfidenceScorer ──► HIGH_CONFIDENCE | UNCERTAIN
                                                 | FALSE_POSITIVE | INSUFFICIENT_EVIDENCE
```

Full request paths and auth planes: [docs/ARCHITECTURE_DIAGRAM.md](docs/ARCHITECTURE_DIAGRAM.md).

---

## Installation

### 1 · Server (Docker)

```bash
# clone the repo
git clone https://github.com/YOUR_GITHUB_USERNAME/cyvex.git
cd cyvex

# set up .env next to docker-compose.yml (untracked - .gitignore covers it):
#   CYVEX_JWT_SECRET=<openssl rand -base64 48>     # >= 32 chars, sessions
#   CYVEX_SIGNING_SECRET=<openssl rand -base64 24> # >= 16 chars, HMAC
#   REDIS_PASSWORD=<random>          # broker password (default: cyvex)
#   POSTGRES_PASSWORD=<random>       # database password (default: cyvex)
#   GOOGLE_API_KEY=...               # optional AI Copilot (omit -> fallback)
# every value has a compose default, so an empty .env still boots - the
# defaults are dev-only: set the secrets before any real deployment.

# build and start the stack
docker compose up -d --build      # postgres, api, worker, redis, gui
docker compose ps                 # all services "Up"
curl http://localhost:8000/health
```

| Service | Port | What it is |
|---|---|---|
| `api` | **8000** | FastAPI — auth, scanner, alerts, reports, licensing |
| `gui` | **8501** | Streamlit ops dashboard |
| `postgres` | internal | tenants, keys, alerts, licenses, audit |
| `redis` + `worker` | internal | Celery continuous scan loop → Slack |

`cyvex server install` can also fetch this repository's
`docker-compose.yml` into `~/.cyvex-server/` and run the stack from
there — see the CLI section below.

### 2 · Web console

```bash
cd frontend
npm install

# frontend/.env.local
#   NEXTAUTH_SECRET=<openssl rand -base64 32>
#   NEXTAUTH_URL=http://localhost:3000
#   FASTAPI_URL=http://localhost:8000

npm run dev                          # http://localhost:3000
```

Register at `/login` (first account of an org becomes **admin**), then
open the dashboard: **Alerts · Scanner · Keys · Billing · Reports ·
Settings · Admin**.

### 3 · Python environment (no Docker)

```bash
python -m venv .venv
# Windows:  .venv\Scripts\activate
# POSIX:    source .venv/bin/activate
pip install -r requirements.txt

python run_pipeline.py --check            # readiness report
python run_pipeline.py --start-api --dev-secrets   # API on :8000
```

---

## CLI usage

Install the CLI (from this repository, editable):

```bash
pip install -e ./cli
```

### First run

```bash
cyvex init        # interactive wizard: org, email, API key,
                  # license activation, start server
cyvex help        # every command + argument, social links at the bottom
```

`cyvex init` writes `./.cyvex` (`api_key`, `base_url`, `org_name`,
`admin_email`, `plan = trial`). There is **no plan prompt**: the
`plan` line is display-only and always starts at `trial`, because the
server decides your tier from your cryptographically verified license,
never from local config - editing the file unlocks nothing. If you
skip license activation the wizard prints the path to a paid tier:
run `cyvex upgrade` to pay via PayPal, then
`cyvex license --activate <key>`.

### Scan and fix

```bash
cyvex scan <path>                 # zip + POST /analyze_repository, render report
cyvex scan <path> --fail-on-high  # exit 1 on HIGH_CONFIDENCE (CI gate)
cyvex fix <file>                  # POST /analyze_code/fix, apply with confirmation
cyvex ci --sarif report.sarif     # SARIF 2.0 for GitHub Code Scanning
cyvex hook install                # .git/hooks/pre-commit running the scan
cyvex report --nca out.pdf        # NCA ECC-2:2024 compliance PDF
```

### Server management

```bash
cyvex server install                  # prompt for a path/URL to docker-compose.yml
cyvex server install ./docker-compose.yml   # verify a local file
cyvex server install https://raw.githubusercontent.com/YOUR_GITHUB_USERNAME/cyvex/main/docker-compose.yml
cyvex server start                    # docker compose up -d, print the URLs
cyvex server stop                     # docker compose down (volumes kept)
cyvex server update                   # pull + up -d
```

The stack lives in `~/.cyvex-server/` (override with
`CYVEX_SERVER_HOME`), so a wheel-only install still gets a full server.
With no argument, `server install` prompts:

```
Enter the path to your docker-compose.yml file or a URL:
```

and an empty answer prints where to place the file manually.

### Licensing (offline, EdDSA)

```bash
cyvex license --request --plan pro   # write cyvex-request-<uuid>.json
                                     # (fingerprint + SHA-256, no network)
cyvex license --activate <file-or-jwt>   # POST /license/activate (X-API-Key)
cyvex license                        # show machine id + usage
```

The license is an Ed25519-signed JWT (`plan`, `mid`, `sub`, `exp`
claims). The API verifies the signature against the on-disk keyring at
startup and again on every protected request: a modified, expired,
unsigned, wrongly-signed or wrong-machine license fails closed. The
`plan` claim — not CLI flags, environment variables, or `.cyvex` — is
what unlocks Pro/Enterprise functionality.

### Purchasing a plan (PayPal)

```bash
cyvex upgrade
```

```
1. Pro Plan ($99/month)
2. Enterprise Plan ($499/month)
```

The selected PayPal link opens in your browser. Complete your payment
on PayPal, run `cyvex license --request` to generate your machine
fingerprint file, and send it to S1lent on Telegram (`@njd0x_s1l3nt`)
or Discord (`p7qo`) to receive your license key. Activate it with
`cyvex license --activate <key>`.

### Social and contact

```bash
cyvex social                # print both community links
cyvex social --discord      # open the Discord invite
cyvex social --telegram     # open the Telegram invite
cyvex contact               # print both direct contacts
cyvex contact --telegram    # print + open t.me/njd0x_s1l3nt
cyvex contact --discord     # print + open discord.com/users/p7qo
```

### Configuration

Resolution order, first hit wins:

```
api key    --api-key   >  CYVEX_API_KEY  >  ./.cyvex  >  ~/.cyvex
base url   --base-url  >  CYVEX_BASE_URL >  ./.cyvex  >  ~/.cyvex
                                            >  http://localhost:8000
```

```bash
cyvex config set-key sk_live_...   # write the key into ./.cyvex
cyvex config show                  # show resolved config (key masked)
```

---

## REST API

| Endpoint | Auth | Purpose |
|---|---|---|
| `POST /auth/register` · `POST /auth/login` | — | org + first admin; JWT |
| `POST /keys/generate` · `GET /keys` · revoke | JWT | `sk_live_` service keys |
| `POST /analyze_code` | **JWT or X-API-Key** | single-file engine report |
| `POST /analyze_repository` | **JWT or X-API-Key** | ZIP → per-file multi-language scan |
| `POST /alert/check` | X-API-Key | CI verdict endpoint (fail-closed) |
| `GET /alerts` | JWT | org-scoped alert queue |
| `GET /reports/nca-compliance` | JWT + admin | NCA ECC-2:2024 PDF |
| `POST /license/request` · `POST /license/activate` | X-API-Key | fingerprint request + license activation |
| `GET /billing/plan` · `GET /billing/features` | JWT | plan + PayPal links, plan feature flags |
| `GET /health` | — | liveness probe |

Error contract everywhere: `{"error": "<code>", "message": "..."}`.

---

## Testing

```bash
python run_pipeline.py --check                 # system readiness
python tests/test_phase39_security.py          # plan authorization + license checks
docker compose config --quiet && echo OK       # compose file valid
cd frontend && npm run typecheck && npm run build
cd vscode-extension && npm run compile
```

---

## References

- **Dong et al., DSN 2025** — up to 50% of security fixes committed with
  no CVE/advisory/changelog; >10% actively exploited before disclosure.
- **Li & Paxson, CCS 2017** — vulnerabilities pre-dating public
  disclosure in open-source components.
- **NCA ECC-2:2024** — Saudi Essential Cybersecurity Controls (v2, 2024);
  Cyvex maps to supply-chain and secure-development controls.

## License

MIT — see [LICENSE](LICENSE).
