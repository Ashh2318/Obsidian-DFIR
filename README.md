<div align="center">

 OBSIDIAN

**AI-powered cybersecurity investigation platform**

`Observe. Analyze. Defend.`

## Overview

OBSIDIAN is an investigation workspace for SOC analysts, DFIR practitioners and
security students. An analyst opens a **case**, uploads **evidence** (packet
captures, logs, documents), and OBSIDIAN runs analysis engines, extracts
indicators of compromise, enriches them with threat intelligence, builds an
event **timeline**, and lets an AI assistant answer questions grounded in the
case data. Findings can be exported as signed executive or technical **reports**.
An optional voice mode ("Hey Obsidian") drives the same actions by speech.

It runs fully offline by default: the AI provider and the threat-intelligence
feeds both have deterministic mock implementations, so no API key, Docker or
database server is needed to try it.

## Features

| Area | What is implemented |
|---|---|
| **Cases** | Create, update, assign; controlled status workflow (open, in progress, contained, closed, archived); severity triage |
| **Evidence** | Upload with size limit, safe file names, server-generated storage paths, content-based type detection, SHA-256 and MD5 hashing, duplicate detection |
| **Analysis engines** | PCAP engine (conversations, protocol distribution, beaconing, port scans, clear-text protocols) and text/log engine (failed logins, encoded PowerShell, persistence indicators), one common result format |
| **IOC extraction** | Twelve indicator types, private-address filtering, defanged view |
| **Threat intelligence** | Mock, AbuseIPDB and VirusTotal providers with a 24-hour cache |
| **Timeline** | Built from the audit trail, evidence metadata and analysis results |
| **AI assistant** | Case-grounded chat with streaming replies; providers: mock, Ollama (local), OpenAI-compatible, Anthropic |
| **Reports** | Executive and technical reports as PDF, HTML and Markdown, signed with HMAC-SHA256 and verifiable |
| **Voice mode** | Wake word, rule-based intent parser and resolver, browser Web Speech API for recognition and spoken replies |
| **Security** | Argon2id, short-lived JWTs, rotating refresh tokens with reuse detection, four-role RBAC, audit log, rate limiting, security headers |

See [`docs/ROADMAP.md`](docs/ROADMAP.md) for what is implemented and what is planned
(EVTX/APK engines, MITRE ATT&CK mapping, retrieval with citations, DOCX export, local Whisper).

## Tech stack

- **Backend:** Python, FastAPI, async SQLAlchemy 2.0, Alembic, Pydantic, pytest
- **Frontend:** Next.js 15 (App Router, PWA), TypeScript, Tailwind CSS, shadcn/ui (Radix)
- **Data:** SQLite for development, PostgreSQL and Redis supported for production
- **Tooling:** GitHub Actions CI, Docker / docker-compose, ruff, ESLint, Prettier, Husky

## Architecture

Clean Architecture with a one-directional dependency rule:

```
routers  ->  services  ->  repositories  ->  database
                 |
                 +------>  AI providers / threat-intel / storage (behind protocols)
```

A router never touches a repository or a session; a service never builds an HTTP
response. Every external dependency (AI, threat intel, cache, storage) sits behind
an interface with an offline mock, so the platform is testable and works without
network access. Voice commands resolve to the same service calls the UI uses.

Details: [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) ·
[`docs/DATABASE_SCHEMA.md`](docs/DATABASE_SCHEMA.md) ·
[`docs/API_REFERENCE.md`](docs/API_REFERENCE.md) ·
[`docs/adr/`](docs/adr/)

## Quick start (Windows)

**Prerequisites:** Python 3.10+ (3.12 recommended, tick *Add python.exe to PATH*) and Node.js 20+.

```bat
git clone https://github.com/<your-username>/obsidian.git
cd obsidian
setup.bat
start.bat
```

`setup.bat` creates the virtual environment, installs dependencies, generates a
fresh `OBSIDIAN_SECRET_KEY`, applies migrations, seeds an admin account and one
demo case, and installs the web dependencies. `start.bat` launches the API and the
web app and opens the browser.

Open **http://localhost:3000** and sign in with the seeded account:

```
email     admin@obsidian.io
password  ChangeMe!2026
```

> Change `OBSIDIAN_SEED_ADMIN_PASSWORD` in `apps/api/.env` before exposing the
> instance to anyone. The first account registered on a fresh deployment also
> becomes administrator.

API docs (Swagger) are at **http://localhost:8000/docs**.

### Manual setup (macOS / Linux)

```bash
# Backend
cd apps/api
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt
pip install -e .
cp .env.example .env
python -c "import secrets; print(secrets.token_urlsafe(64))"   # paste into OBSIDIAN_SECRET_KEY
alembic upgrade head
python seed.py --demo
uvicorn obsidian.main:app --reload --port 8000

# Frontend (second terminal)
cd apps/web
npm install
cp .env.local.example .env.local
npm run dev
```

### Docker

```bash
docker compose up --build
```

## Configuration

All backend settings are environment variables prefixed `OBSIDIAN_`
(see [`apps/api/.env.example`](apps/api/.env.example)).

| Variable | Purpose |
|---|---|
| `OBSIDIAN_SECRET_KEY` | JWT and report-signing secret, 32+ characters (required) |
| `OBSIDIAN_DATABASE_URL` | Defaults to a local SQLite file |
| `OBSIDIAN_AI_PROVIDER` | `mock` (default, offline), `ollama`, `openai`, `anthropic` |
| `OBSIDIAN_AI_MODEL` | Model name only, for example `gpt-oss:20b` |
| `OBSIDIAN_*_API_KEY` | Only for the provider you enable; never commit these |
| `OBSIDIAN_THREATINTEL_PROVIDERS` | `mock` by default; `abuseipdb,virustotal` need keys |

`configure.bat` walks through provider and key setup and backs up your `.env` first.
Use **Ollama** when evidence legally cannot leave the machine.

### Troubleshooting

| Script | Use |
|---|---|
| `doctor.bat` | Diagnoses an API that will not start, including a port-8000 clash (Splunk and Jenkins use it) |
| `configure.bat` | Guided AI / threat-intel key setup |
| `test.bat` | Backend tests plus frontend type check |

## Tests

```bash
cd apps/api
pytest -v
```

The backend suite (301 tests at the last verified run) covers authentication,
token rotation and reuse detection, RBAC, case and evidence workflows, the PCAP and
text engines, threat-intel enrichment, timeline, assistant, reporting and voice.
CI runs lint, type check, tests, a fresh-database migration check and dependency
audits on every push.

## Project layout

```
OBSIDIAN/
├── apps/
│   ├── api/    FastAPI service  (core, db, domain, repositories, services,
│   │           analysis, threatintel, ai, reporting, voice, api/v1)
│   └── web/    Next.js 15 PWA   (app routes, components, hooks, lib)
├── docs/       architecture, API reference, schema, roadmap, ADRs
├── scripts/    setup helpers, secret scanner, docs generator
└── .github/    CI workflow
```

## Security

Argon2id password hashing; 15-minute access tokens held in memory only; opaque,
hashed, single-use refresh tokens with family revocation on reuse; account lockout
and per-IP rate limiting; uniform login responses to prevent enumeration; four
ordered roles; append-only audit trail; HSTS, CSP, X-Frame-Options and related
headers. Run `python scripts/check_secrets.py` before every push.

Please do not open public issues for vulnerabilities; contact the author directly.

## Design references

The design draws on published work, including Fielding's REST dissertation (API
style), NIST SP 800-61 Rev. 2 (incident handling lifecycle), the ALCE benchmark and
"Lost in the Middle" (grounded, citation-aware use of retrieved context),
indirect prompt-injection research (treating evidence as untrusted input to the
assistant), GenDFIR and the LLM forensic-timeline evaluation work (timeline
analysis), CTI2TTP and ThreatPilot (MITRE ATT&CK extraction), and Whisper (planned
local speech recognition). Several of these map to planned rather than finished
features; the roadmap states which.

## Roadmap

Planned: EVTX, document and APK engines; ATT&CK technique mapping;
retrieval with citations; Sigma/YARA rule generation; DOCX export and approval
workflow; local Whisper and Piper voice; Nginx profile and desktop packaging.
Full plan in [`docs/ROADMAP.md`](docs/ROADMAP.md).

## License

Copyright © 2026 Aastha Rawal. All rights reserved. The source may be viewed for
evaluation (for example academic assessment or recruitment review). Copying,
modifying, redistributing or reusing it requires written permission. See
[`LICENSE`](LICENSE). Third-party libraries keep their own licences.

## Author

**Aastha Rawal**
