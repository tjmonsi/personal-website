# Personal Website — Spec-Driven Development in Action

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
![Status](https://img.shields.io/badge/Status-Specifications_Phase-orange)
![Specs](https://img.shields.io/badge/Specs-7_documents-green)
![Clarifications](https://img.shields.io/badge/Clarification_Rounds-14-purple)

> **No code has been written yet.** This repository is a real-world demonstration of Spec-Driven Development (SDD) — the practice of writing comprehensive specifications *before* writing a single line of implementation code.

## What This Project Is

This is the specification repository for [TJ Monserrat's](https://github.com/tjmonsi) personal website. It will eventually be a full-stack web application with a technical blog, opinion blog, social links, and curated content — but right now, the entire repository is **documentation only**.

The real value here isn't the website itself — it's the **process**. This repository captures how a senior engineer approaches software design: removing assumptions, documenting decisions, and iterating on specifications with AI assistance until they're implementation-ready.

## Quick Start (How to Use This Repository)

If you're new here, follow this path:

1. Read [`archive/plans/initial-plan.md`](archive/plans/initial-plan.md) to see the raw starting point.
2. Read the 7 spec files in [`specs/`](specs/) from `01` to `07`.
3. Read [`tasks.md`](tasks.md) to see how specs map to implementation work.
4. Read [`HISTORY.md`](HISTORY.md) for the repository evolution with commit-by-commit context.

This repository is currently documentation-only. No application code has been implemented yet.

### The Tech Stack (Planned)

| Layer | Technology |
|-------|-----------|
| Frontend | Nuxt 4 / Vue 3 / Vite (SPA mode) |
| Backend | Go on Google Cloud Run |
| Database | Firestore Enterprise (MongoDB compatibility) |
| Search | Vector search via Firestore Native + Vertex AI embeddings |
| Infrastructure | GCP — Cloud Run, Cloud Armor, Cloud Load Balancer, Firebase Hosting |
| IaC | Terraform |
| Observability | Cloud Logging, Cloud Monitoring, BigQuery, Looker Studio |

## Why This Exists

### The Gap Between Coding and Software Engineering

There's a steep skill jump from writing code to engineering software. Many developers jump straight into implementation — open an IDE, start coding, figure it out as they go. That works for small scripts and tutorials, but it falls apart for production systems where you need to think about:

- Security (CSP headers, rate limiting, JWT authentication, CORS)
- Infrastructure (VPCs, load balancers, IAM, service accounts)
- Observability (structured logging, SLIs/SLOs, alerting)
- Cost (GCP pricing, free tiers, traffic spike modeling)
- Data modeling (schema design, vector indexes, retention policies)
- Edge cases (what happens when a banned user's IP is shared by an ISP?)

**Spec-Driven Development forces you to confront all of these decisions before you write code**, when changing direction is free. A change in a spec document is a text edit. A change in production code can be a week of refactoring.

### What You'll Learn From This Repository

1. **How to write specs that are implementation-ready** — unambiguous, testable, with acceptance criteria
2. **How to use AI as a specification partner** — using Claude Opus 4.6 with the SE: Architect agent and reusable prompts to identify gaps, inconsistencies, and missing edge cases
3. **How iterative clarification works** — 14 rounds of clarification questions and answers, refining the specs from a rough plan into precise engineering documents
4. **What "removing all assumptions" looks like in practice** — documented decisions for everything from JWT delivery mechanisms (`sendBeacon` vs `fetch`) to SLO rolling window durations (28-day vs 30-day)

### The Advantage of Domain Knowledge

The author's background in cloud computing, distributed systems, networking (TCP, REST, request-response), parallel computing, race conditions, memory management, and client-server architecture made it possible to answer clarification questions precisely and make informed trade-off decisions. This is the kind of knowledge that separates someone who can write code from someone who can design systems:

- Knowing that `navigator.sendBeacon()` can't send custom HTTP headers — so JWT auth needs to go in the request body
- Understanding that a single IP might serve many users behind an ISP's NAT — so rate limiting needs nuance
- Recognizing that CSP `img-src 'self'` will block images in markdown articles served from a different domain
- Choosing 2048-dimension vectors over 768 because the embedding model outputs them natively

**This is something a developer shifting from "coder" to "software engineer" will need to learn — and reading through the git history of this project is one way to see that thought process in action.**

## Repository Structure

```
├── specs/                          # The specification documents (start here)
│   ├── 01-system-overview.md       # Architecture, tech stack, high-level design
│   ├── 02-frontend-specifications.md   # Pages, components, UX behavior
│   ├── 03-backend-api-specifications.md # Endpoints, request/response contracts
│   ├── 04-data-model-specifications.md  # Database schemas, collections, indexes
│   ├── 05-infrastructure-specifications.md # GCP resources, Terraform, networking
│   ├── 06-security-specifications.md    # Auth, CSP, CORS, rate limiting, IAM
│   └── 07-observability-specifications.md # Logging, monitoring, SLIs/SLOs, alerts
│
├── docs/
│   └── cost-estimate-draft.md      # GCP monthly cost modeling (2 scenarios)
│
├── clarifications/                 # Active clarification questions (current round)
│
├── archive/
│   ├── plans/
│   │   └── initial-plan.md         # The original rough plan that started everything
│   └── previous-clarifications/    # All 13 completed clarification rounds
│       ├── clarifications-2026-02-17-0001.md
│       ├── ...
│       └── clarifications-2026-02-17-0013.md
│
├── .github/
│   ├── prompts/                    # Reusable AI prompts (update-specification, etc.)
│   ├── agents/                     # Custom AI agent definitions
│   ├── instructions/               # Copilot instruction files
│   └── skills/                     # Copilot skill definitions
│
└── LICENSE                         # Apache 2.0
```

## How to Read This Project

### If you want to understand the final specs:

Read the files in `specs/` in order (01 through 07), then the `docs/cost-estimate-draft.md`.

### If you want to see how the specs evolved:

```bash
# Clone the repository
git clone https://github.com/tjmonsi/personal-website.git
cd personal-website

# See the full commit history
git log --oneline

# Read a summarized explanation of the same history
cat HISTORY.md

# See how a specific spec file changed over time
git log --oneline -- specs/06-security-specifications.md

# See the diff for a specific commit
git show <commit-hash>
```

The evolution follows a clear pattern:

1. **Initial plan** → rough feature list in plain English
2. **First spec generation** → AI transforms plan into 7 structured spec documents
3. **Clarification rounds 1–13** → AI identifies gaps, owner answers, specs are updated
4. **Architecture additions** → vector search, Terraform IaC, WIF auth, BigQuery analytics
5. **Acceptance criteria** → testable Given-When-Then criteria added to every spec
6. **Ongoing refinement** → round 14 in progress

### If you want to see the AI collaboration workflow:

Look at the `archive/previous-clarifications/` folder. Each file shows:
- Questions the AI asked (with options and context)
- The owner's answers (with reasoning)
- How those answers were applied to the specs

The reusable prompts in `.github/prompts/` — especially [update-specification.prompt.md](.github/prompts/update-specification.prompt.md) — show how to structure AI interactions for specification work.

## The Workflow

This is the process used to create these specifications, completed in roughly **one day** with AI assistance:

```
 ┌─────────────────────────────────────────────────────┐
 │  1. Write initial plan (human domain knowledge)     │
 └──────────────────────┬──────────────────────────────┘
                        ▼
 ┌─────────────────────────────────────────────────────┐
 │  2. AI generates structured specs from plan         │
 │     → 7 spec files + cost estimate                  │
 └──────────────────────┬──────────────────────────────┘
                        ▼
 ┌─────────────────────────────────────────────────────┐
 │  3. AI reviews specs, asks clarification questions  │◄──┐
 │     → Identifies gaps, inconsistencies, ambiguities │   │
 └──────────────────────┬──────────────────────────────┘   │
                        ▼                                   │
 ┌─────────────────────────────────────────────────────┐   │
 │  4. Human answers with domain expertise             │   │
 │     → Makes trade-off decisions, adds constraints   │   │
 └──────────────────────┬──────────────────────────────┘   │
                        ▼                                   │
 ┌─────────────────────────────────────────────────────┐   │
 │  5. AI applies answers, updates all affected specs  │   │
 │     → Cross-references, bumps versions, archives    │   │
 └──────────────────────┬──────────────────────────────┘   │
                        ▼                                   │
 ┌─────────────────────────────────────────────────────┐   │
 │  6. AI re-checks for new gaps                       ├───┘
 │     → If gaps found, go to step 3                   │
 │     → If none, human reviews the specs              │
 └──────────────────────┬──────────────────────────────┘
                        ▼
 ┌─────────────────────────────────────────────────────┐
 │  7. Human reviews, adds requirements (e.g., IaC,    │
 │     navigation, acceptance criteria)                 │
 │     → Go back to step 3 for AI review               │
 └─────────────────────────────────────────────────────┘
```

## Tools Used

- **AI Model**: Claude Opus 4.6 via GitHub Copilot
- **Agent**: SE: Architect (custom agent — see `.github/agents/`)
- **Reusable Prompts**: `update-specification.prompt.md` and others in `.github/prompts/`
- **Instructions**: Custom Copilot instructions for code review, security, performance, and more in `.github/instructions/`
- **Editor**: VS Code with GitHub Copilot

## Current Status

| Artifact | Version | Status |
|----------|---------|--------|
| System Overview | v3.7 | ✅ Reviewed |
| Frontend Specs | v3.7 | ✅ Reviewed |
| Backend API Specs | v4.1 | ✅ Reviewed |
| Data Model Specs | v3.5 | ✅ Reviewed |
| Infrastructure Specs | v4.8 | ✅ Reviewed |
| Security Specs | v4.7 | ✅ Reviewed |
| Observability Specs | v4.1 | ✅ Reviewed |
| Cost Estimate | v1.2-draft | ✅ Reviewed |
| Clarification Round 14 + Gap Analysis v1-v11 | — | ✅ Completed |

**Next step**: Start implementation planning and execute the tasks in [`tasks.md`](tasks.md).

## Contributing

This is a personal project, but if you're interested in the Spec-Driven Development approach, feel free to:

- Open an issue with questions about the process
- Fork the repo structure to use as a template for your own SDD projects
- Look at the `.github/prompts/` and `.github/instructions/` folders for reusable AI prompt patterns

## License

This project is licensed under the Apache License 2.0 — see the [LICENSE](LICENSE) file for details.
