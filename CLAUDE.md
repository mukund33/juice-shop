# OWASP Juice Shop — AI Assistant Reference & DevSecOps Guide

This document is the primary reference for AI assistants (Claude and others) working on this codebase. It covers architecture, development workflows, conventions, and the integrated DevSecOps pipeline.

---

## Repository Overview

**OWASP Juice Shop** (`v19.1.1`) is an intentionally vulnerable web application for security training and CTFs. It is a full-stack TypeScript/Angular application with 100+ security challenges built in.

> **Important for AI Assistants**: Many dependencies and code patterns here are **intentionally vulnerable**. Do NOT "fix" security issues unless explicitly asked. The vulnerabilities ARE the product. See [Intentionally Vulnerable Components](#intentionally-vulnerable-components).

---

## Codebase Architecture

```
juice-shop/
├── app.ts                    # Application entry point
├── server.ts                 # Express server setup (~36k lines)
├── tsconfig.json             # TypeScript config (target: ES2020, outDir: ./build)
├── Dockerfile                # Multi-stage build → distroless runtime image
├── docker-compose.test.yml   # Smoke test composition
├── cypress.config.ts         # E2E test config (baseUrl: localhost:3000)
├── Gruntfile.js              # Packaging tasks
├── swagger.yml               # API documentation
├── bom.json / bom.xml        # Generated CycloneDX SBOMs (gitignored)
│
├── frontend/                 # Angular 20 SPA (separate npm project)
│   ├── package.json          # Frontend deps (Karma, Jasmine, Angular Material)
│   ├── angular.json          # CLI configuration
│   ├── src/                  # Components, services, assets
│   └── eslint.config.js      # Flat ESLint config with Angular rules
│
├── routes/                   # Express route handlers (~50 files)
│   ├── login.ts              # JWT auth (intentionally weak)
│   ├── search.ts             # SQL injection challenge
│   ├── basket.ts             # Shopping cart logic
│   ├── fileUpload.ts         # File upload challenge
│   ├── redirect.ts           # Open redirect challenge
│   ├── metrics.ts            # Prometheus metrics endpoint
│   └── ...
│
├── models/                   # Sequelize ORM models (~18 entities)
│   ├── user.ts               # User model (SQLite3 backend)
│   ├── product.ts            # Product model
│   ├── challenge.ts          # Challenge tracking
│   └── ...
│
├── lib/                      # Utilities and library functions
│   ├── insecurity.ts         # Security utilities (JWT, hashing) — intentionally weak
│   ├── challengeUtils.ts     # Challenge state management
│   ├── utils.ts              # General utilities
│   ├── logger.ts             # Winston logging setup
│   └── startup/              # 6 startup modules (validation, customization)
│
├── data/                     # Static data and challenge definitions
│   ├── datacreator.ts        # Database seeding
│   ├── static/challenges.yml # All challenge definitions
│   ├── static/codefixes/     # 50+ code fix tutorials (excluded from linting/SAST)
│   └── chatbot/              # Chatbot training data
│
├── test/                     # All test suites
│   ├── api/                  # Jest/frisby API integration tests
│   ├── cypress/e2e/          # 31 Cypress E2E specs
│   ├── server/               # Mocha server unit tests
│   └── smoke/                # Shell script smoke tests
│
├── config/                   # 12 environment configs (YAML)
│   ├── default.yml           # Default config
│   ├── ctf.yml               # CTF competition mode
│   ├── unsafe.yml            # All challenges enabled
│   └── ...
│
├── monitoring/               # Grafana dashboard configs
├── rsn/                      # Refactoring Safety Net scripts
├── .zap/rules.tsv            # ZAP scan ignore rules (known acceptable issues)
├── .github/workflows/        # 11 GitHub Actions CI/CD pipelines
└── .well-known/              # security.txt, CSAF advisories
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| `server.ts` | Core Express app: middleware, routes, socket.io, Prometheus |
| `lib/insecurity.ts` | JWT creation/validation, hashing — intentionally uses weak algorithms |
| `lib/challengeUtils.ts` | Challenge solve tracking and notification |
| `routes/login.ts` | Auth endpoint with intentional bypass vulnerabilities |
| `routes/search.ts` | Product search with intentional SQL injection |
| `data/static/challenges.yml` | Master challenge registry |
| `config/default.yml` | Default application configuration schema |

---

## Intentionally Vulnerable Components

These are **by design** and must NOT be "fixed" without an explicit challenge modification request:

| Package | Vulnerability Purpose |
|---------|----------------------|
| `express-jwt@0.1.3` | Weak JWT validation |
| `jsonwebtoken@0.4.0` | Algorithm confusion attacks |
| `sanitize-html@1.4.2` | XSS bypass |
| `unzipper@0.8.14` | Path traversal |

These are pinned in `.dependabot/config.yml` under `ignore:`.

---

## Development Commands

```bash
# Install all dependencies (runs postinstall: frontend build + server compile)
npm install

# Development server (ts-node + Angular serve, with hot reload)
npm run serve:dev

# Production build + start
npm run build:server && npm start

# Lint (ESLint on *.ts + frontend lint + SCSS lint)
npm run lint
npm run lint:fix

# Validate YAML config schemas
npm run lint:config

# Unit tests (frontend Karma/Jasmine + server Mocha)
npm test

# API integration tests (Jest + frisby)
npm run frisby               # or: npm run test:api

# E2E tests (Cypress)
npm run cypress:run          # headless
npm run cypress:open         # interactive UI

# Refactoring Safety Net (run after touching challenge code)
npm run rsn
npm run rsn:update           # update cache after intentional changes

# SBOM generation (CycloneDX JSON + XML)
npm run sbom                 # generates bom.json and bom.xml
npm run sbom:json
npm run sbom:xml

# Packaging
npm run package              # Grunt packaging
npm run package:ci           # Production package with SBOM generation
```

---

## Testing Strategy

| Test Type | Framework | Command | Coverage |
|-----------|-----------|---------|---------|
| Frontend unit | Karma + Jasmine | `npm test` | Angular components/services |
| Server unit | Mocha + nyc | `npm run test:server` | lib/, models/, routes/ |
| API integration | Jest + frisby | `npm run frisby` | All REST endpoints |
| E2E | Cypress 13 | `npm run cypress:run` | 31 challenge flows |
| Smoke | Shell script | `test/smoke/` | Post-package validation |

Coverage reports land in `./build/reports/coverage/` (lcov + text-summary).

---

## Code Style

- **Language**: TypeScript (strict mode, ES2020 target)
- **Linter**: ESLint with `standard-with-typescript`
- **Style guide**: [JS Standard Style](http://standardjs.com/)
- **Frontend linter**: Angular ESLint flat config + Prettier
- **SCSS linter**: stylelint-config-sass-guidelines
- Always run `npm run lint` before committing.
- The `data/static/codefixes/` directory is **excluded** from linting (intentionally bad code).

---

## Git Workflow

```bash
# Always sign off commits (DCO requirement)
git commit -s -m "feat: description"

# Feature branches from develop
git checkout -b feat/your-feature develop

# PR targets: develop branch
```

---

## DevSecOps Pipeline

The repository integrates a full DevSecOps pipeline across four security pillars, with all results aggregated into a unified dashboard. The pipeline runs on every push/PR and weekly.

### Pipeline Overview

```
Push / PR / Schedule
        │
        ▼
┌───────────────────────────────────────────────────┐
│                 GitHub Actions                     │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐│
│  │   SBOM   │  │   SCA    │  │      SAST        ││
│  │CycloneDX │  │ npm audit│  │ CodeQL + Semgrep ││
│  │bom.json  │  │ OWASP DC │  │ SARIF → GitHub   ││
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘│
│       │             │                  │           │
│       └─────────────┼──────────────────┘           │
│                     │                              │
│  ┌──────────────────┼────────────────────────────┐ │
│  │        DAST: OWASP ZAP                        │ │
│  │  Dockerized Juice Shop → ZAP Active Scan      │ │
│  └──────────────────┬────────────────────────────┘ │
└─────────────────────┼──────────────────────────────┘
                      │ All scan artifacts
                      ▼
              ┌───────────────┐
              │   DefectDojo  │  ← Unified Security Dashboard
              │  (self-hosted │
              │   or cloud)   │
              └───────────────┘
              GitHub Security tab also
              receives all SARIF reports
```

---

### 1. SBOM — Software Bill of Materials

**Tool**: [CycloneDX](https://cyclonedx.org/) (already integrated)

```bash
npm run sbom          # generates bom.json + bom.xml
npm run sbom:json     # JSON format only
npm run sbom:xml      # XML format only
```

- Runs automatically during `npm run package:ci`
- Also generated inside the Dockerfile during image build
- Excludes devDependencies (`--omit=dev`)
- Outputs: `bom.json`, `bom.xml` in project root

**In CI** (`.github/workflows/devsecops.yml`): SBOM is generated, uploaded as artifact, and imported into DefectDojo as a `CycloneDX Scan`.

---

### 2. SCA — Software Composition Analysis

**Tools**: npm audit + OWASP Dependency Check

```bash
# npm audit (fast, uses npm advisory DB)
npm audit --json > reports/npm-audit.json

# OWASP Dependency Check (comprehensive NVD CVE scan)
dependency-check.sh \
  --project "juice-shop" \
  --scan . \
  --format ALL \
  --out reports/dependency-check/ \
  --suppression dependency-check-suppressions.xml
```

**Suppression file** (`dependency-check-suppressions.xml`): Suppresses findings for intentionally vulnerable packages (express-jwt, jsonwebtoken, sanitize-html, unzipper). This is intentional — do not remove these suppressions.

**In CI**: SCA runs on every push. npm audit JSON is imported into DefectDojo as `NPM Audit Scan`. OWASP DC XML is imported as `OWASP Dependency Check`.

---

### 3. SAST — Static Application Security Testing

**Tools**: GitHub CodeQL (existing) + Semgrep (new)

#### CodeQL (`.github/workflows/codeql-analysis.yml`)
- Runs on every push and PR
- Language: `javascript-typescript`
- Query suite: `security-extended`
- Excludes: `data/static/codefixes/`
- Results appear in GitHub Security > Code scanning tab

#### Semgrep (`.github/workflows/devsecops.yml`)
```bash
# Local Semgrep scan
docker run --rm \
  -v "$(pwd):/src" \
  semgrep/semgrep scan \
  --config "p/javascript" \
  --config "p/nodejs" \
  --config "p/owasp-top-ten" \
  --config "p/typescript" \
  --sarif \
  --output reports/semgrep.sarif \
  --exclude "data/static/codefixes/**" \
  --exclude "node_modules/**"
```

Semgrep SARIF is uploaded to both:
1. GitHub Security tab (via `github/codeql-action/upload-sarif`)
2. DefectDojo (as `Semgrep JSON Report`)

**Expected SAST findings in Juice Shop**: Semgrep and CodeQL will flag many intentional vulnerabilities (SQL injection in `routes/search.ts`, XSS, hardcoded secrets in `lib/insecurity.ts`). These are marked as accepted risks in DefectDojo.

---

### 4. DAST — Dynamic Application Security Testing

**Tool**: OWASP ZAP

#### Existing weekly scan (`.github/workflows/zap_scan.yml`)
- Schedule: Saturday 18:00 UTC
- Target: `https://preview.owasp-juice.shop`
- Mode: Baseline scan (`-a -j`)
- Ignore rules: `.zap/rules.tsv`

#### Enhanced CI scan (`.github/workflows/devsecops.yml`)
- Spins up `bkimminich/juice-shop` as a Docker service
- Runs ZAP baseline against `http://localhost:3000`
- Generates: `report_json.json`, `report_html.html`, `report_md.md`
- Results imported into DefectDojo as `ZAP Scan`

```bash
# Local ZAP scan against running Juice Shop
docker run --rm \
  -v "$(pwd)/.zap:/zap/wrk/:rw" \
  -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py \
  -t http://host.docker.internal:3000 \
  -r zap-report.html \
  -J zap-report.json \
  -c rules.tsv
```

**ZAP ignore rules** (`.zap/rules.tsv`): ~20 known acceptable findings are suppressed (CSP headers not set, HSTS, etc.) because Juice Shop intentionally disables many protections for challenge purposes.

---

### 5. Unified Dashboard — DefectDojo

**Tool**: [DefectDojo](https://github.com/DefectDojo/django-DefectDojo) (open source)

DefectDojo is the central security findings dashboard. It aggregates results from all four pillars, deduplicates findings across tools, and tracks remediation status.

#### Local Setup

```bash
# Start DefectDojo with Docker Compose
cd defectdojo/
docker compose up -d

# Access UI
open http://localhost:8080
# Default credentials: admin / defectdojo
# Change immediately: http://localhost:8080/api/key-v2
```

DefectDojo configuration is in `defectdojo/docker-compose.yml`. It runs:
- `nginx` reverse proxy
- `uwsgi` Django app
- `celery` worker + beat
- `redis` message broker
- `postgres` database

#### CI Integration (Secrets Required)

Add these secrets to GitHub repository settings:

| Secret | Description |
|--------|-------------|
| `DEFECTDOJO_URL` | Base URL of your DefectDojo instance (e.g., `https://defectdojo.example.com`) |
| `DEFECTDOJO_API_TOKEN` | API token from DefectDojo (Profile > API v2 Key) |

If `DEFECTDOJO_URL` is not set, the pipeline still runs but skips DefectDojo upload steps.

#### DefectDojo Structure

```
Organization: OWASP
  └── Product: Juice Shop
        └── Engagement: DevSecOps CI/CD - <branch>
              ├── Test: CycloneDX Scan (SBOM)
              ├── Test: NPM Audit Scan (SCA)
              ├── Test: OWASP Dependency Check (SCA)
              ├── Test: Semgrep JSON Report (SAST)
              └── Test: ZAP Scan (DAST)
```

#### Importing Reports Manually

```bash
# Upload any scan report to DefectDojo via API
curl -X POST "http://localhost:8080/api/v2/import-scan/" \
  -H "Authorization: Token <your-token>" \
  -F "scan_type=ZAP Scan" \
  -F "file=@zap-report.json" \
  -F "product_name=Juice Shop" \
  -F "engagement_name=Manual Import" \
  -F "auto_create_context=true"

# Supported scan_type values used in this project:
# "CycloneDX Scan"          → bom.json (SBOM)
# "NPM Audit Scan"          → npm-audit.json (SCA)
# "OWASP Dependency Check"  → dependency-check-report.xml (SCA)
# "Semgrep JSON Report"     → semgrep.sarif (SAST)
# "ZAP Scan"                → report_json.json (DAST)
```

#### GitHub Security Tab (Lightweight Alternative)

All SARIF-formatted results (CodeQL, Semgrep, ZAP) are automatically uploaded to the GitHub **Security > Code scanning** tab — no additional setup required. This provides a basic unified view within GitHub without needing DefectDojo.

---

## CI/CD Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `ci.yml` | push/PR | Full test matrix (lint, unit, API, E2E, smoke, Docker) |
| `devsecops.yml` | push/PR/weekly | SBOM + SCA + SAST (Semgrep) + DAST (ZAP) + DefectDojo |
| `codeql-analysis.yml` | push/PR | GitHub CodeQL SAST |
| `zap_scan.yml` | Saturday 18:00 UTC | ZAP baseline on preview instance |
| `release.yml` | tags | Release packaging and publishing |
| `lint-fixer.yml` | PR | Automated lint fixes |

---

## AI Assistant Guidelines

### Allowed Tasks

- Code analysis, refactoring, test writing, bug fixing, documentation
- Adding new route handlers, models, or utility functions
- Writing Cypress E2E tests or frisby API tests
- Updating CI/CD workflows
- Security review and DevSecOps tooling

### Requires Caution

- **New challenges**: Consult maintainers — AI-generated challenges risk being duplicate or broken
- **Dependency updates**: Many deps are intentionally vulnerable; check `.dependabot/config.yml` ignore list
- **Architecture changes**: Discuss with maintainers first
- **Translations**: Use Crowdin, never edit i18n files directly

### Commit Requirements

```bash
# Always sign off (DCO)
git commit -s -m "type: description"
```

### Quality Checklist Before Committing

- [ ] `npm run lint` passes (ESLint + frontend lint + SCSS)
- [ ] `npm run lint:config` passes (YAML schema validation)
- [ ] Relevant tests added and passing
- [ ] `npm run rsn` passes (if touching challenge code)
- [ ] No verbose AI-generated comments added
- [ ] Commits are signed off (`-s` flag)
- [ ] PR targets `develop` branch

### Anti-Patterns

| Don't | Do |
|-------|----|
| "Fix" intentional vulnerabilities | Understand what's intentional |
| Accept AI code without understanding it | Review every line |
| Add verbose explaining comments | Write no comments unless WHY is non-obvious |
| Skip tests because AI "seems confident" | Run the full test suite |
| Modify translations in files | Use Crowdin |
| Suppress security findings without reason | Document why in DefectDojo |
| Update pinned vulnerable packages | Check `.dependabot/config.yml` first |

---

## Refactoring Safety Net (RSN)

When modifying code used in coding challenges:

```bash
npm run rsn          # check for unexpected changes
npm run rsn:update   # update cache after intentional changes
npm run rsn:verbose  # detailed output
```

RSN validates that `data/static/codefixes/` snippets match the actual source. Required before committing any change that touches challenge-referenced code.

---

## Environment Configurations

| Config file | Use case |
|------------|---------|
| `config/default.yml` | Standard development |
| `config/ctf.yml` | CTF competition (score server integration) |
| `config/unsafe.yml` | All challenges enabled (used in CI) |
| `config/tutorial.yml` | Guided tutorial mode |
| `config/mozilla.yml` / `config/7ms.yml` | Custom event configs |

Set active config: `NODE_ENV=ctf npm start`

---

## Monitoring

Prometheus metrics endpoint: `GET /metrics`

Grafana dashboards: `monitoring/` directory contains pre-built dashboard JSON configs.

Local Prometheus + Grafana stack: not included in this repo; connect to `/metrics` endpoint manually.

---

## Getting Help

- Contribution guidelines: [CONTRIBUTING.md](CONTRIBUTING.md)
- Full pwning guide: https://pwning.owasp-juice.shop
- Community: Slack / Gitter (links in CONTRIBUTING.md)
- Security issues: [SECURITY.md](SECURITY.md) (PGP-encrypted reports)
- GitHub issues: https://github.com/juice-shop/juice-shop/issues
