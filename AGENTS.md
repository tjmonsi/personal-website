# AGENTS.md

## Project Overview

Personal website for TJ Monserrat — a full-stack web application with a technical blog, opinion blog, social links, and curated external content. Currently in the **specifications phase** — no implementation code has been written yet. All 7 specification documents are finalized and implementation-ready.

### Architecture

| Layer | Technology | Runtime |
|-------|-----------|---------|
| Frontend | Nuxt 4 / Vue 3 / Vite (SPA mode) | Firebase Hosting + Firebase Functions |
| Backend API | Go 1.22+ / Fiber v2 | Cloud Run (`asia-southeast1`) |
| Database | Firestore Enterprise (MongoDB compat) | GCP (`asia-southeast1`) |
| Vector Search | Firestore Native Mode + Vertex AI `gemini-embedding-001` | GCP (`asia-southeast1`) |
| Cloud Functions | Node.js 22 LTS (Gen 2) × 4 | GCP (`asia-southeast1`) |
| Infrastructure | Terraform | GCS remote state |
| CDN / WAF | Cloud Load Balancer + Cloud Armor | GCP |
| Observability | Cloud Logging + Cloud Monitoring + BigQuery + Looker Studio | GCP |

### Domain

- Frontend: `tjmonsi.com` (Firebase Hosting)
- Backend API: `api.tjmonsi.com` (Cloud Run behind Load Balancer + Cloud Armor)

---

## Repository Structure

```
├── AGENTS.md                       # This file — agent context
├── CODE_STYLE_GUIDE.md             # Coding standards for Python, JS/TS, and Go
├── README.md                       # Human-facing project overview
├── tasks.md                        # 160-task implementation plan (7 phases)
├── specs/                          # Finalized specification documents
│   ├── 01-system-overview.md       # Architecture, tech stack, decisions
│   ├── 02-frontend-specifications.md   # Pages, components, UX behavior
│   ├── 03-backend-api-specifications.md # 13 endpoints, middleware, errors
│   ├── 04-data-model-specifications.md  # 12 Firestore collections, indexes
│   ├── 05-infrastructure-specifications.md # GCP resources, Terraform, CI/CD
│   ├── 06-security-specifications.md    # Auth, CSP, CORS, rate limiting, IAM
│   └── 07-observability-specifications.md # Logging, monitoring, SLIs/SLOs
├── manuals/
│   └── pre-requisite-manual.md     # GCP bootstrap guide (before Terraform)
├── docs/
│   └── cost-estimate-draft.md      # GCP monthly cost modeling
├── .github/
│   ├── agents/                     # 16 custom AI agent definitions
│   ├── instructions/               # 33 Copilot instruction files
│   ├── prompts/                    # 12 reusable AI prompts
│   └── skills/                     # 3 Copilot skills (excalidraw, git-commit, refactor)
├── clarifications/                 # Active clarification round (if any)
└── archive/                        # Completed clarification rounds and initial plan
```

### Planned Project Structure (Implementation)

When implementation begins, the following directories will be created:

```
├── terraform/                      # Terraform IaC for all GCP infrastructure
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── backend.tf
│   └── versions.tf
├── backend/                        # Go backend API (Fiber v2)
│   ├── cmd/api/main.go
│   ├── internal/
│   │   ├── handler/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── model/
│   │   ├── middleware/
│   │   └── router/
│   ├── go.mod
│   ├── go.sum
│   ├── Dockerfile
│   └── .golangci.yml
├── frontend/                       # Nuxt 4 SPA
│   ├── nuxt.config.ts
│   ├── firebase.json
│   ├── eslint.config.mjs
│   ├── pages/
│   ├── components/
│   ├── composables/
│   ├── layouts/
│   ├── public/
│   └── tests/
└── functions/                      # Cloud Functions Gen 2 (Node.js 22)
    ├── generate-sitemap/
    ├── process-rate-limit-logs/
    ├── cleanup-rate-limit-offenders/
    └── sync-article-embeddings/
```

---

## Implementation Plan

The full implementation plan is in [tasks.md](tasks.md) — 160 tasks across 7 phases:

| Phase | Tasks | Description |
|-------|-------|-------------|
| P0 | 10 | Manual bootstrap (GCP project, SAs, WIF, secrets) |
| P1 | 39 | Terraform infrastructure (VPC, Firestore, Cloud Run, etc.) |
| P2 | 49 | Go backend API (middleware, endpoints, vector search) |
| P3 | 7 | Cloud Functions Gen 2 (sitemap, rate limit, embeddings) |
| P4 | 34 | Nuxt 4 frontend SPA (pages, components, offline support) |
| P5 | 11 | CI/CD pipelines and initial deployment |
| P6 | 5 | Observability dashboards |
| P7 | 5 | Content pipeline integration |

**Parallelization**: Phases 2, 3, and 4 can run in parallel after Phase 1. Phase 4 can start with mocked APIs.

---

## Specification Reference

When implementing any feature, **always consult the relevant spec first**. Each spec uses numbered identifiers (e.g., `BE-API-002`, `DM-004`, `INFRA-008c`, `SEC-013`) that are cross-referenced throughout.

| Spec | Key Identifiers | What to Check |
|------|----------------|---------------|
| [01-system-overview.md](specs/01-system-overview.md) | AD-001 through AD-023 | Architectural decisions and rationale |
| [02-frontend-specifications.md](specs/02-frontend-specifications.md) | FE-PAGE-001–010, FE-COMP-001–014 | Page behavior, component specs, UX rules |
| [03-backend-api-specifications.md](specs/03-backend-api-specifications.md) | BE-API-001–013 | Endpoint contracts, middleware chain, error format |
| [04-data-model-specifications.md](specs/04-data-model-specifications.md) | DM-001–012 | Collection schemas, indexes, constraints |
| [05-infrastructure-specifications.md](specs/05-infrastructure-specifications.md) | INFRA-001–020 | GCP resources, Terraform, CI/CD pipelines |
| [06-security-specifications.md](specs/06-security-specifications.md) | SEC-001–014 | Auth, headers, CORS, IAM, rate limiting |
| [07-observability-specifications.md](specs/07-observability-specifications.md) | OBS-001–011 | Logging, monitoring, alerts, SLIs/SLOs |

---

## Setup Commands

### Prerequisites (One-Time Manual)

Follow [manuals/pre-requisite-manual.md](manuals/pre-requisite-manual.md) for the full bootstrap guide. Summary:

```bash
# Enable required GCP APIs
gcloud services enable \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  storage.googleapis.com \
  cloudbilling.googleapis.com

# Create Terraform state bucket
gsutil mb -l asia-southeast1 -b on gs://${PROJECT_ID}-terraform-state
gsutil versioning set on gs://${PROJECT_ID}-terraform-state
```

### Terraform

```bash
cd terraform/
terraform init
terraform plan -var="project_id=${PROJECT_ID}" -var="region=asia-southeast1"
terraform apply -var="project_id=${PROJECT_ID}" -var="region=asia-southeast1"
```

### Go Backend

```bash
cd backend/
go mod download
go run cmd/api/main.go  # Local development

# Testing
go test ./...
go test -race ./...
go test -cover ./...

# Linting
golangci-lint run
gofmt -l .
go vet ./...

# Docker build
docker build -t website-api:local .
```

### Nuxt Frontend

```bash
cd frontend/
npm install  # or pnpm install
npm run dev  # Start dev server with hot reload

# Testing
npx vitest
npx vitest --coverage

# Linting
npx eslint .
npx nuxi typecheck

# Build for production
npx nuxt build --preset=firebase
```

### Cloud Functions

```bash
cd functions/<function-name>/
npm install
npm test

# Deploy (done via CI/CD, but manual if needed)
gcloud functions deploy <function-name> \
  --gen2 \
  --runtime=nodejs22 \
  --region=asia-southeast1 \
  --trigger-<type>
```

### Local Development

```bash
# Start local MongoDB (for Firestore Enterprise compat)
docker-compose up -d mongodb

# Start Firestore emulator (for Native Mode vector search)
gcloud emulators firestore start --host-port=localhost:8080

# Run backend against local databases
MONGODB_URI=mongodb://localhost:27017 \
FIRESTORE_EMULATOR_HOST=localhost:8080 \
go run cmd/api/main.go
```

---

## Code Style

Detailed conventions are in [CODE_STYLE_GUIDE.md](CODE_STYLE_GUIDE.md). Key points per language:

### Go (Backend API)

- **Formatter**: `gofmt` (tabs, canonical formatting)
- **Linter**: `golangci-lint` (config in `.golangci.yml`)
- **Naming**: `MixedCaps` for exports, `mixedCaps` for internal, short names for short scopes
- **Constants**: `MixedCaps` (NOT `SCREAMING_SNAKE_CASE`)
- **Errors**: Always check, wrap with `fmt.Errorf("context: %w", err)`, return early
- **Testing**: Table-driven tests with `t.Run`, file alongside source (`user.go` → `user_test.go`)
- **Framework**: Fiber v2 — handlers return `error`, use `c.UserContext()` for context propagation
- **Project layout**: `cmd/api/main.go`, `internal/{handler,service,repository,model,middleware,router}/`

### JavaScript / TypeScript (Frontend + Cloud Functions)

- **Indentation**: 4 spaces
- **Semicolons**: None (Neostandard default)
- **Quotes**: Single quotes
- **Trailing commas**: Always on multiline
- **Linter**: ESLint 9 flat config with Neostandard + `@stylistic`
- **Types**: TypeScript strict mode, no `any`, prefer `interface` for objects
- **Vue**: `<script setup lang="ts">`, auto-imports (don't import `ref`/`computed`/`useRoute`), PascalCase components
- **Testing**: Vitest, `*.test.ts` or `*.spec.ts`, Arrange-Act-Assert pattern

### Python (Utility Scripts)

- **Line length**: 100 characters
- **Formatter**: Black
- **Linter**: Ruff + mypy
- **Types**: Full annotations required, modern syntax (`X | None`, `list[str]`)
- **Docstrings**: Google-style

### Git Commits

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`

---

## Testing Instructions

### Go Backend

```bash
cd backend/

# Run all tests
go test ./...

# Run tests with race detector
go test -race ./...

# Run tests with coverage
go test -cover ./...

# Run specific test
go test -run TestCreateUser ./internal/service/

# Run with verbose output
go test -v ./...
```

- Test files: `*_test.go` alongside source files
- Naming: `TestFunctionName_Scenario_ExpectedResult`
- Pattern: Table-driven tests with `t.Run` subtests
- Mocking: Define interfaces at call site, implement test doubles

### Nuxt Frontend

```bash
cd frontend/

# Run all tests
npx vitest

# Run in watch mode
npx vitest --watch

# Run with coverage
npx vitest --coverage

# Run specific test file
npx vitest tests/components/SearchBar.test.ts

# Type checking
npx nuxi typecheck
```

- Test files: `*.test.ts` or `*.spec.ts` in `tests/` directory
- Framework: Vitest
- Pattern: Arrange-Act-Assert with descriptive sentence-style names

### Cloud Functions

```bash
cd functions/<function-name>/

# Run tests
npm test

# Run with coverage
npx vitest --coverage
```

### CI Pipeline Checks

All of these must pass before merge:

```bash
# Go
golangci-lint run
go test ./...

# Frontend
npx eslint .
npx vitest
npx nuxi typecheck

# Cloud Functions
npx eslint .
npm test

# Security scans
govulncheck ./...    # Go vulnerabilities
npm audit            # Node.js vulnerabilities
trivy image <image>  # Docker image scan
```

---

## Build and Deployment

### Docker Image (Go Backend)

Multi-stage build: `golang:1.24-alpine` → `gcr.io/distroless/static-debian12`

```bash
docker build -t website-api:$(git rev-parse --short HEAD) .
docker push asia-southeast1-docker.pkg.dev/${PROJECT_ID}/website-images/website-api:$(git rev-parse --short HEAD)
```

### Firebase (Frontend)

```bash
npx nuxt build --preset=firebase
firebase deploy --only hosting,functions
```

### CI/CD Pipelines

Two GitHub Actions workflows:

1. **`terraform.yml`** (INFRA-019): `fmt` → `validate` → `plan` → `apply` (manual approval). Auth: WIF → `terraform-builder@` SA.
2. **`deploy.yml`** (INFRA-020): `lint` → `test` → `build` → `scan` → `deploy`. Auth: WIF → `app-deployer@` SA. Deploys backend Docker image, frontend via Firebase, and 4 Cloud Functions.

Both pipelines authenticate via **Workload Identity Federation** — no long-lived service account keys.

---

## Environment Configuration

| Environment | Database | Networking |
|-------------|----------|------------|
| Development | Local MongoDB + Firestore Emulator | No GCP resources, no VPC |
| Production | Firestore Enterprise + Firestore Native | VPC with Direct VPC Egress |

### Key Environment Variables

```bash
# Backend (Cloud Run)
GCP_PROJECT_ID          # GCP project identifier
MONGODB_URI             # Firestore Enterprise connection string
FIRESTORE_PROJECT_ID    # For Firestore Native SDK
MAXMIND_DB_PATH         # Path to GeoLite2-Country.mmdb in container
JWT_SECRET              # HMAC-SHA256 signing key for POST /t

# CI/CD (GitHub Actions secrets)
GCP_PROJECT_ID
WIF_PROVIDER            # projects/<num>/locations/global/workloadIdentityPools/<pool>/providers/<provider>
TERRAFORM_SA_EMAIL      # terraform-builder@<project>.iam.gserviceaccount.com
APP_DEPLOYER_SA_EMAIL   # app-deployer@<project>.iam.gserviceaccount.com
MAXMIND_LICENSE_KEY     # For downloading GeoLite2 database
```

---

## Security Considerations

- **No hardcoded secrets** — all secrets via environment variables or GCP Secret Manager
- **API is GET-only** with a single `POST /t` exception for visitor tracking
- **Cloud Armor** rate limits at 60 req/min per IP with progressive banning (3 offenses → 30d, 4 → 90d, 5+ → indefinite)
- **IP anonymization** — IPv4 last octet zeroed, IPv6 last 80 bits zeroed before logging
- **Security headers** on every response: CSP, HSTS, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy
- **CORS** restricted to `tjmonsi.com` origin only
- **JWT** for tracking payload integrity — HMAC-SHA256, 5-min expiry, 60s clock skew tolerance
- **10 service accounts** with least-privilege IAM roles (see SEC-013)
- **WIF authentication** for all CI/CD pipelines — no long-lived keys

---

## Debugging and Troubleshooting

### Common Issues

- **Firestore Enterprise vs Native**: Two separate databases. Enterprise uses MongoDB Go driver (`go.mongodb.org/mongo-driver`). Native uses Firestore Go SDK (`cloud.google.com/go/firestore`). Don't mix them up.
- **Vector search dimensions**: Must be exactly 2048 (Gemini `gemini-embedding-001` with `output_dimensionality=2048`). Firestore Native max is 2048.
- **Content hash**: `content_hash` in DM-002/DM-003 covers stored fields only (`title`, `slug`, `category`, `abstract`, `tags`, `content`, `changelog`). Excludes dynamically generated citations and system fields.
- **Cloud Armor logs**: Rate limit events (429) flow through Cloud Logging → Pub/Sub → Cloud Function, not directly to the backend.
- **Firebase preset**: Nuxt must be built with `--preset=firebase` for Firebase Hosting + Functions deployment.

### Logging

- **Backend**: Structured JSON to stdout (Cloud Run captures automatically). Fields: `timestamp`, `severity`, `request_id`, `method`, `path`, `status_code`, `latency_ms`, `geo_country`, `truncated_ip`.
- **Frontend**: Errors sent via `POST /t` with `error_report` action type. In-memory breadcrumb trail (50 entries) attached to error reports.
- **BigQuery**: 6 log sinks route logs to `website_logs` dataset for long-term analysis.

### Useful Commands

```bash
# Check Cloud Run logs
gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="website-api"' --limit=50

# Check Cloud Function logs
gcloud logging read 'resource.type="cloud_run_revision" AND resource.labels.service_name="<function-name>"' --limit=50

# Check Cloud Armor decisions
gcloud logging read 'resource.type="http_load_balancer" AND jsonPayload.enforcedSecurityPolicy.name="<policy>"' --limit=50

# Firestore Enterprise (MongoDB shell)
mongosh "mongodb://<connection-string>" --eval "db.technical_articles.countDocuments()"

# Terraform state
cd terraform/
terraform state list
terraform state show <resource>
```

---

## PR Guidelines

- **Title format**: `<type>(<scope>): <description>` (Conventional Commits)
- **Required checks before submit**: All lint, test, and typecheck commands must pass
- **Commit messages**: Follow Conventional Commits format
- **Branch strategy**: `dev` for development, merge to `main` for production deployment
- **One concern per PR**: Keep PRs focused — separate infrastructure, backend, frontend, and Cloud Function changes where practical

---

## Additional Notes

- The **content pipeline** (articles, frontpage, socials) lives in a separate repository and is out of scope. Only the integration contract (INFRA-014) is specified here.
- The **spec-driven workflow** means all implementation must trace back to a spec identifier. When implementing a feature, reference the relevant spec ID in commit messages and code comments.
- **Two Firestore databases** exist in the same GCP project: Enterprise (MongoDB compat, default) and Native (`vector-search`). They serve different purposes and use different client libraries.
- The project uses **no third-party analytics** — all visitor tracking is self-hosted via `POST /t` → Cloud Logging → BigQuery → Looker Studio.
- **MaxMind GeoLite2** database is embedded in the Docker image and refreshed weekly via a separate CI workflow.
