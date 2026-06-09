# AI Code Reviewer — Complete Project Reference (`ALLINFO`)

> A single, exhaustive reference for understanding, explaining, maintaining, demoing, and interviewing on this project.
>
> **Inferred project name:** **AI Code Reviewer** (canonical slug used throughout the infra: `ai-code-reviewer`).
> **Repository directory:** `github-PR-reviewer-AI-Agent`.
> **Evidence basis:** Everything below is derived from the tracked files in this repository (64 tracked files, clean working tree at time of writing). Where the repo does not provide enough evidence, this document says so explicitly with the phrase *"Unclear from repo evidence."* No features, metrics, or integrations have been invented.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Metadata](#2-project-metadata)
3. [Quick Start Guide](#3-quick-start-guide)
4. [What The Project Does](#4-what-the-project-does)
5. [High-Level Architecture](#5-high-level-architecture)
6. [End-to-End Workflows](#6-end-to-end-workflows)
7. [Data Model, State, And Configuration](#7-data-model-state-and-configuration)
8. [API, Routes, Commands, And Entrypoints](#8-api-routes-commands-and-entrypoints)
9. [Full Repository Map](#9-full-repository-map)
10. [File-By-File Deep Dive](#10-file-by-file-deep-dive)
11. [Cross-Cutting Concerns](#11-cross-cutting-concerns)
12. [Testing And Validation](#12-testing-and-validation)
13. [Build, Deployment, And Operations](#13-build-deployment-and-operations)
14. [How To Modify Or Extend This Project](#14-how-to-modify-or-extend-this-project)
15. [Interview Preparation Pack](#15-interview-preparation-pack)
16. [Glossary](#16-glossary)
17. [Risks, Gaps, And Improvement Roadmap](#17-risks-gaps-and-improvement-roadmap)
18. [Coverage Checklist](#18-coverage-checklist)

---

## 1. Executive Summary

### One-paragraph overview

**AI Code Reviewer** is a GitHub App that automatically reviews pull requests using four single-purpose LLM agents running in parallel, packaged as a fleet of independently deployable microservices on AWS EKS (Kubernetes). When a developer opens (or reopens / pushes to) a PR on a repository where the App is installed, GitHub sends a webhook; the system verifies it, persists the PR, fetches the diff, fans it out to four role-specialized agents (static analysis, security, style, architecture) that each call `gpt-4o-mini`, merges and de-duplicates their findings, and posts a single consolidated review back onto the PR — typically within about a minute. When a PR is *merged*, a "learner" extracts the recurring `warning`/`error` findings into a per-repository pattern memory that conditions the *style* agent's prompt on future reviews. The whole thing is wired with LLM tracing (LangFuse), a scheduled offline evaluation (RAGAS), per-service CI/CD (GitHub Actions over OIDC), metrics (Prometheus/Grafana), and Infrastructure-as-Code (Terraform).

### The problem it solves

Human code review is a bottleneck, and the "boring but important" defects — a hardcoded secret, an `eval()` over user input, a missing error path — are precisely the ones a tired reviewer skims past. This bot performs an automatic first pass the instant a PR opens, so a human reviewer starts from a triaged list of findings rather than a blank diff.

### Target users

- **Primary:** Software teams that want an automated first-pass reviewer on their GitHub PRs.
- **Actual / honest framing (per `README.md`):** This is a **portfolio / case-study project** built to demonstrate end-to-end ownership of an LLM feature — the multi-agent design, the surrounding backend, and the cloud/MLOps work to ship it. It was deployed to AWS, validated on a single real PR, then torn down with `terraform destroy` so ongoing cost is **$0**. There is **no permanently-live demo by design**.

### The core workflow

```
PR opened ─▶ HMAC verify ─▶ persist + enqueue ─▶ fetch diff ─▶ 4 agents in parallel
          ─▶ merge/dedupe ─▶ persist findings ─▶ post ONE consolidated GitHub review
PR merged ─▶ enqueue learning ─▶ extract warning/error findings ─▶ upsert per-repo patterns
```

### The main technical achievement

A **LangGraph multi-agent orchestration** whose conditional entry point fans a diff out to four role-specific agents concurrently (`Send` API), each returning structured JSON, followed by a merge/dedupe node — running as **seven Kubernetes workloads** (5 HTTP services + 2 Celery workers) with full cloud infrastructure in Terraform, per-service CI/CD, autoscaling, tracing, and offline evaluation. Per the author's own note, the genuinely new ground was the *cloud/MLOps* stack (EKS, IAM/OIDC, ECR, ALB ingress, Prometheus/Grafana) rather than the model call itself.

### How I would pitch this in an interview (condensed)

> "I built an AI pull-request reviewer as a production-shaped system, not a notebook. A GitHub App webhook hits an edge gateway that does HMAC verification only; a webhook service persists the PR and drops a job on a Redis/Celery queue so the webhook can return `202` immediately; a Celery worker calls an orchestrator that mints a GitHub App installation token, pulls the diff, and runs a LangGraph graph that fans out to four specialized `gpt-4o-mini` agents in parallel. Their findings are merged, written to Postgres, and a reviewer service posts one consolidated review — with a fallback to a summary comment when LLM-supplied line numbers don't anchor to the diff. On merge, a learner distills recurring issues into a per-repo pattern table that conditions future reviews. Every model call is traced in LangFuse, a weekly RAGAS job gates on faithfulness, and all seven services ship through per-service GitHub Actions pipelines that authenticate to AWS over OIDC and deploy to EKS. All infra — VPC, EKS, RDS, ElastiCache, ECR, S3 — is Terraform. I deployed it, validated it on a real PR for a fraction of a cent, then tore it down to $0."

---

## 2. Project Metadata

| Field | Value (from repo evidence) |
|---|---|
| **Inferred project name** | AI Code Reviewer (`ai-code-reviewer`) — from `README.md` title, and consistent `ai-code-reviewer` naming in Terraform/k8s/GitHub App setup. |
| **Repository type** | Polyglot mono-repo of microservices + IaC + CI/CD + docs. Not a published package. |
| **Main language** | Python 3.11 (all services). |
| **Secondary "languages"** | HCL (Terraform), YAML (Kubernetes, GitHub Actions, Prometheus), Dockerfile, JSON (Grafana), Markdown, INI (Alembic). |
| **Frameworks / libraries** | FastAPI, Uvicorn, SQLAlchemy (async), Alembic, Pydantic / pydantic-settings, Celery, Redis client, httpx, PyJWT + cryptography, LangGraph, OpenAI SDK, LangFuse, RAGAS, `datasets`, psycopg2, `prometheus-fastapi-instrumentator`. |
| **Package / build tools** | `pip` + per-service `requirements.txt` (unpinned except the evaluate image); Docker (one image per service); Terraform; Helm (for AWS LB Controller, Prometheus, Grafana — invoked operationally, not tracked as charts). |
| **Runtime assumptions** | Runs on AWS: EKS (Kubernetes 1.32), RDS PostgreSQL 15, ElastiCache Redis 7.0, ECR, S3, an AWS ALB via the AWS Load Balancer Controller. Local dev is **not** a first-class path (see §3). |
| **External services / integrations** | GitHub (App auth + webhooks + REST API), OpenAI (`gpt-4o-mini`), LangFuse (tracing), AWS (compute/data/registry), Prometheus + Grafana (metrics). |
| **Important entrypoints** | `services/<svc>/main.py` (FastAPI ASGI apps via `uvicorn main:app`); `services/webhook/worker.py` & `services/learner/worker.py` (Celery apps); `scripts/evaluate.py` (RAGAS batch job); `db/migrations/env.py` (Alembic). |
| **Important config files** | `infra/k8s/configmap.yaml`, `infra/k8s/secret.yaml` (placeholder), `db/alembic.ini`, `monitoring/prometheus.yml`, `monitoring/grafana-dashboard.json`, `infra/terraform/*.tf`, each `services/<svc>/models.py` `Settings` class, `.gitignore`. |
| **Test commands found** | In `.github/workflows/service-ci.yml`: `if [ -d tests ]; then pytest; fi` — i.e. `pytest` runs **only if** a `services/<svc>/tests/` dir exists. None do, so no tests execute today. |
| **Run / build commands found** | Per Dockerfile: `uvicorn main:app --host 0.0.0.0 --port 800X`. Workers: `celery -A worker worker --loglevel=info -Q <queue>`. Build: `docker build -f services/<svc>/Dockerfile .`. |
| **Deployment clues found** | `infra/terraform/` (full AWS stack), `infra/k8s/` (Deployments/Services/HPA/Ingress/Jobs), `.github/workflows/` (5 per-service pipelines + reusable CI + weekly eval), `INSTRUCTIONS.md` (16-phase runbook), `services/*/deploy.txt` (redeploy triggers), `git log` (deployment-themed commits). |
| **License** | MIT, `Copyright (c) 2026 Srikara S` (`LICENSE`). |

---

## 3. Quick Start Guide

> ⚠️ **Reality check (from `README.md` "How to run" and `INSTRUCTIONS.md`):** this project is designed to run **entirely on AWS**. Docker images are built by **GitHub Actions**, not locally, and there is **no `docker-compose.yml`, no local dev script, and no `.env.example`** tracked in the repo. Your machine only needs CLIs (`terraform`, `kubectl`, `helm`, `aws`, `git`). A purely-local run is **not a supported path in the repo** — see "What's missing for local dev" below.

### Prerequisites

- An **AWS account** (the system provisions VPC, EKS, RDS, ElastiCache, ECR, S3).
- A **GitHub App** with a webhook secret and an RSA private key, plus a GitHub repo with Actions enabled.
- An **OpenAI API key** (`gpt-4o-mini` access).
- **LangFuse** keys (public `pk-lf-…` / secret `sk-lf-…`).
- CLIs installed: `terraform`, `kubectl`, `helm`, `aws`, `git` (the runbook also uses the AWS CLI for OIDC/IAM setup).

### Install / provision steps (the supported "cloud-first" path)

Condensed from `README.md` §"How to run" and the 16 phases of `INSTRUCTIONS.md`:

1. **Provision infra.** From `infra/terraform`: `terraform init` then `terraform apply -var="cluster_name=ai-code-reviewer" -var="db_password=…" -var="environment=…"`. Creates VPC, EKS (k8s 1.32), RDS Postgres 15, ElastiCache Redis 7.0, **6** ECR repos (gateway/webhook/orchestrator/reviewer/learner/evaluate), S3 bucket, IAM policy for the LB controller. Default region `ap-southeast-2`.
2. **Configure GitHub → AWS OIDC deploys.** Create an IAM OIDC provider + role (`github-actions-ai-reviewer`), then add three **repo secrets**: `AWS_ROLE_ARN`, `AWS_ACCOUNT_ID`, `EKS_CLUSTER_NAME`. (No long-lived AWS keys; GitHub assumes the role over OIDC.)
3. **Build & push.** Pushing to `main` triggers the five per-service pipelines (path-filtered): test → build image → push to ECR → `kubectl set image`. On a *clean* cluster the deploy job fails until step 4 has created the Deployments, then re-trigger.
4. **Apply k8s manifests** (exact order in `INSTRUCTIONS.md` §15): `configmap.yaml`, `secret.yaml` (filled from a gitignored local copy), `migration-job.yaml` (Alembic), the 5 services + 2 workers, `hpa.yaml`, install the **AWS Load Balancer Controller** via Helm, then `ingress.yaml`. The manifests reference a literal `${ECR_REGISTRY}` placeholder — substitute your registry at apply time.
5. **Point the GitHub App at the ALB.** Webhook URL = `http://<alb-address>/webhook/github`.
6. **Open a PR** on an installed repo → the bot reviews it.

### Environment variables (names only — values redacted)

These are consumed by the `Settings` classes (`pydantic-settings`) in each service and supplied via the k8s ConfigMap/Secret. **Never commit real values** (see `.gitignore` and the placeholder `secret.yaml`).

| Variable | Used by | Secret? | Notes |
|---|---|---|---|
| `GITHUB_WEBHOOK_SECRET` | gateway | **secret** `[REDACTED]` | HMAC-SHA256 key for webhook verification. |
| `GITHUB_APP_ID` | orchestrator, reviewer | secret `[REDACTED]` | GitHub App ID; `iss` claim of the JWT. |
| `GITHUB_APP_PRIVATE_KEY` | orchestrator, reviewer | **secret** `[REDACTED]` | RSA key (RS256) used to mint installation tokens. `\n` escapes are un-escaped at runtime. |
| `OPENAI_API_KEY` | orchestrator, evaluate | **secret** `[REDACTED]` | Read implicitly by the OpenAI SDK. |
| `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` | orchestrator | **secret** `[REDACTED]` | LangFuse tracing creds. |
| `LANGFUSE_HOST` | orchestrator | config | `https://cloud.langfuse.com` (EU) or `…us.cloud…` (US). |
| `DATABASE_URL` | webhook, orchestrator, reviewer, learner, migrate, evaluate | **secret** `[REDACTED]` | `postgresql+asyncpg://…` for services; `evaluate.py` rewrites it to plain `postgresql://` for psycopg2. |
| `REDIS_URL` | webhook(+worker), learner-worker | config | Celery broker/back-end. ⚠️ the tracked `configmap.yaml` contains a **real-looking ElastiCache hostname** (internal, now torn down) — see §11. |

### Development / build / test commands (as found)

- **Run a service locally (manual, not scripted):** `cd services/<svc> && pip install -r requirements.txt && uvicorn main:app --port 800X`. You must still provide a reachable Postgres + Redis + the env vars; there is no fixture for these.
- **Build an image:** `docker build -t <svc> -f services/<svc>/Dockerfile .` (build context is the repo root; webhook image also copies `db/`).
- **Run a worker:** `celery -A worker worker --loglevel=info -Q webhook` (or `-Q learning`).
- **Run migrations:** `alembic -c db/alembic.ini upgrade head` with `DATABASE_URL` set.
- **Tests:** `pytest` — but see "troubleshooting" below; there are **no test files**, so this is a no-op.

### Common troubleshooting (distilled from `INSTRUCTIONS.md` §23)

| Symptom | Likely cause / fix |
|---|---|
| Pod `Pending` forever | `Insufficient cpu/memory` (raise node `desired_size`) or `ImagePullBackOff` (image not in ECR — re-run build-and-push). |
| Pod `CrashLoopBackOff` | Bad `DATABASE_URL` (must start `postgresql+asyncpg://`), missing secret key, or a Python import error (`kubectl logs POD --previous`). |
| Webhook deliveries fail `404` | Webhook URL must end with `/webhook/github`. |
| Webhook deliveries fail `401` | `GITHUB_WEBHOOK_SECRET` mismatch between GitHub App and k8s secret. |
| No review comments appear | Invalid/zero-credit OpenAI key, missing GitHub App "Pull requests: Read & write", or malformed private key (line breaks lost). |
| `terraform apply` fails "VPC limit exceeded" | AWS default is 5 VPCs/region — delete an old one or request an increase. |
| `kubectl` "must be logged in" | kubeconfig token expired — re-run `aws eks update-kubeconfig …`. |

### What's missing for a clean local dev experience (gaps, stated honestly)

- **No `docker-compose.yml`** to bring up Postgres + Redis + all services together.
- **No `.env.example`** (it's allow-listed in `.gitignore` but not actually committed).
- **No seed/fixtures** and **no tests** to validate a local setup.
- Service-to-service URLs are **hardcoded to k8s DNS names** (`http://webhook:8001`, `http://orchestrator:8002`, `http://reviewer:8003`, `http://learner:8004`) in the Python, so running outside the cluster requires either matching hostnames (e.g. `/etc/hosts`, compose service names) or code edits.

---

## 4. What The Project Does

### Main user-facing features

1. **Automatic PR review.** On `opened` / `reopened` / `synchronize`, the bot posts **one consolidated GitHub review** (`event: "COMMENT"`) containing a markdown summary and, when possible, line-anchored inline comments.
2. **Four review dimensions.** Each finding is tagged with the agent that produced it:
   - **static_analysis** — complexity, unused variables, poor naming.
   - **security** — OWASP Top 10, hardcoded secrets, SQL injection risks.
   - **style** — formatting, readability, consistency (this agent is *conditioned on learned patterns*).
   - **architecture** — separation-of-concerns, missing error handling, improper dependency usage.
3. **Per-repository learning.** On merge, recurring `warning`/`error` findings are distilled into a `patterns` table; the top-10 most frequent patterns for a repo are injected into the *style* agent's prompt on subsequent PRs — a lightweight feedback loop (no fine-tuning).
4. **Graceful inline-comment fallback.** If GitHub rejects the inline anchors with HTTP `422` (LLM line numbers don't always match the diff), the reviewer retries with just the summary body so the review still posts.

### Main developer-facing / operator-facing features

- **Per-service CI/CD** (5 path-filtered pipelines sharing one reusable workflow) with OIDC auth to AWS.
- **LLM observability:** every model call traced in LangFuse (prompt, response, tokens, cost, latency).
- **Offline evaluation:** a weekly RAGAS job (faithfulness + answer-relevancy) that *fails* below 0.7 mean faithfulness.
- **Metrics:** every HTTP service exposes `/metrics`; a Grafana dashboard tracks per-service request rate, p50/p99 latency, 5xx rate, plus an overall-health row.
- **Autoscaling:** an HPA scales the orchestrator (2→6 replicas) on 70% CPU.
- **IaC + runbook:** Terraform for the whole AWS stack and a 1,881-line `INSTRUCTIONS.md` step-by-step guide.

### Inputs and outputs

| | Input | Output |
|---|---|---|
| **System (whole)** | GitHub PR webhook (JSON event + `X-Hub-Signature-256`) | A posted GitHub PR review (summary + optional inline comments); rows in Postgres; LangFuse traces; Prometheus metrics. |
| **gateway** | Raw webhook body + signature header | Forwarded request to `webhook` (or `401`). |
| **webhook** | Parsed event JSON | `pull_requests` row + enqueued Celery task; `202`/`skipped`/`already_processing`. |
| **orchestrator** | `AnalyzeRequest` (pr_id, pr_number, repo, head_sha, installation_id) | `findings` rows + a call to the reviewer; `202`. |
| **graph (LangGraph)** | `{diff, patterns, findings:[]}` | `findings` list of `{file,line,severity,message,agent}`. |
| **reviewer** | `ReviewRequest` (findings + PR identity) | GitHub review posted; PR `status="reviewed"`. |
| **learner** | `LearnRequest` (repo, pr_id) | Upserted `patterns` rows. |
| **evaluate** | Last 50 findings from Postgres | RAGAS scores printed; exit `1` if faithfulness < 0.7. |

### Important endpoints / commands / jobs

- HTTP: `POST /webhook/github` (gateway), `POST /events` (webhook), `POST /analyze` (orchestrator), `POST /post-review` (reviewer), `POST /learn` (learner), `GET /health` + `GET /metrics` (all).
- Queues: `webhook` (task `analyze_pr`), `learning` (task `trigger_learning`).
- Jobs: `db-migrate` (Alembic), `evaluate` (RAGAS, weekly cron `0 9 * * 1` + manual dispatch).

### Expected happy path

PR opened → `200` from gateway after HMAC passes → `202` from webhook after row+enqueue → Celery `analyze_pr` → orchestrator mints token, fetches diff, runs 4 agents in parallel, writes findings → reviewer posts one review → PR marked `reviewed`. End-to-end ≈ "about a minute" (README), dominated by the slowest agent (~5s median model latency, agents concurrent).

### Important failure paths

- **Bad signature** → gateway `401`, nothing downstream runs.
- **Non-PR / irrelevant action** → webhook returns `skipped`.
- **Duplicate `(repo, pr_number, head_sha)`** → webhook returns `already_processing` (no second review).
- **Empty findings** → reviewer returns early without posting.
- **GitHub `422` on inline anchors** → reviewer retries summary-only.
- **GitHub API / OpenAI errors** → `raise_for_status()` / exceptions propagate; the Celery task (with `timeout=120`) is the outermost guard. There is **no explicit retry/back-off or dead-letter handling** in the task code.

### Known limitations visible from the code (corroborated by `README.md`)

1. **Findings are emitted twice.** `GraphState.findings` uses an `operator.add` reducer; `merge_node` returns the deduped list which is then *appended* to the raw findings instead of replacing them → duplicates persisted and posted. (See §6 and §10 `graph.py`.)
2. **Inline comments often fall back to summary** because LLM line numbers aren't grounded against the diff.
3. **No automated tests** exist (CI runs `pytest` only if a `tests/` dir is present; none is).
4. **The RAGAS evaluation is wired but unproven** (needs a real body of reviewed PRs).
5. **A likely latent bug in the learning loop's Celery wiring** — the `trigger_learning` task body is defined in `services/webhook/worker.py` but the `learner-worker` (which consumes the `learning` queue) runs `services/learner/worker.py`, which does **not** define that task. See §6, §10, and §17.
6. **Cost-tuned, not hardened infra:** public subnets, single-AZ DB, HTTP (not HTTPS) ingress.

---

## 5. High-Level Architecture

### Major components (7 Kubernetes workloads + data/infra)

| Component | Port / role | Kind |
|---|---|---|
| **gateway** | `:8000` — HMAC-SHA256 webhook verification only; forwards verified bodies | FastAPI Deployment (2 replicas) + `ClusterIP` + `LoadBalancer` Service |
| **webhook** | `:8001` — persist PR, dedupe, enqueue Celery task; handle merge events | FastAPI Deployment (2) + `ClusterIP` |
| **webhook-worker** | Celery worker on queue `webhook`; calls orchestrator | Deployment (2) |
| **orchestrator** | `:8002` — GitHub App token, diff fetch, LangGraph 4-agent run, persist findings, call reviewer | FastAPI Deployment (2, **HPA 2→6**) + `ClusterIP` |
| **reviewer** | `:8003` — build summary + inline comments, post one GitHub review, mark PR reviewed | FastAPI Deployment (2) + `ClusterIP` |
| **learner** | `:8004` — on merge, extract findings → upsert per-repo patterns | FastAPI Deployment (2) + `ClusterIP` |
| **learner-worker** | Celery worker on queue `learning` | Deployment (1) |
| **PostgreSQL 15 (RDS)** | `pull_requests`, `findings`, `patterns` | Managed data store |
| **Redis 7 (ElastiCache)** | Celery broker + result backend | Managed queue |
| **ECR / S3 / ALB / EKS** | image registry / report bucket / ingress / cluster | AWS infra (Terraform) |

> Note on the "seven services" framing: the repo contains **five HTTP services** plus **two Celery workers** (which reuse the webhook/learner images with a different `command`). That is the "seven" in "seven microservices." The `evaluate` image is a sixth ECR repo / batch Job, not a long-running service.

### How they communicate

- **Inbound:** GitHub → AWS ALB (ingress) → `gateway` Service → pod.
- **East-west:** plain HTTP over Kubernetes DNS service names (`http://webhook:8001`, `http://orchestrator:8002`, `http://reviewer:8003`, `http://learner:8004`). Hardcoded in the Python.
- **Async decoupling:** `webhook` → Redis (Celery) → `webhook-worker`/`learner-worker` → orchestrator/learner over HTTP.
- **Outbound:** orchestrator/reviewer → GitHub REST API; orchestrator → OpenAI (via LangFuse drop-in client) → LangFuse.

### Frontend / backend split

There is **no frontend**. The "UI" is the GitHub PR itself (where reviews appear) plus operator dashboards (Grafana, Prometheus, LangFuse, GitHub Actions). All tracked code is backend / infra.

### Database / storage layer

- **PostgreSQL** via async SQLAlchemy (`asyncpg`), schema created by Alembic migration `0001_initial`. Three tables (see §7). `pgcrypto` extension enabled for `gen_random_uuid()`.
- **Redis** as the Celery broker/back-end.
- **S3** bucket provisioned for "reports" (declared in Terraform; no tracked code writes to it — *Unclear from repo evidence whether it is used at runtime*).

### Authentication / authorization

- **Inbound auth:** HMAC-SHA256 over the raw webhook body, constant-time compared (`hmac.compare_digest`) against `X-Hub-Signature-256` (gateway).
- **Outbound auth to GitHub:** GitHub **App** flow — an RS256 JWT (`iss=app_id`, `iat=now-60`, `exp=now+600`) is exchanged at `/app/installations/{id}/access_tokens` for a short-lived installation token used to read the diff and post the review.
- **AWS auth (CI/CD):** GitHub Actions assumes an IAM role via **OIDC** (no stored AWS keys); EKS access entry grants that role cluster-admin.

### AI / ML / LLM / agentic components

- **LangGraph** `StateGraph` with a **conditional entry point** (`fan_out`) that emits four `Send` commands → four agent nodes run concurrently → all converge on a `merge` node → `END`.
- Each agent = one `gpt-4o-mini` chat completion with a role-specific **system prompt** + the **diff** as the user message; output parsed as a JSON array (with a fenced-code-block fallback).
- The **style** agent's prompt is dynamically built to include the repo's learned patterns.
- **LangFuse** wraps the OpenAI client (`from langfuse.openai import OpenAI`) so every call is traced.
- **RAGAS** (offline) scores findings for faithfulness + answer relevancy.

### Background workers / jobs

- Celery workers (`webhook-worker`, `learner-worker`).
- Kubernetes Jobs: `db-migrate` (run once at deploy), `evaluate` (weekly cron via GitHub Actions, which `kubectl apply`s the Job).

### External API / service calls

GitHub REST (`api.github.com`), OpenAI (`gpt-4o-mini`), LangFuse cloud, AWS APIs (provisioning/deploy).

### Error handling / logging / observability

- HTTP errors: `raise_for_status()` on outbound calls; `HTTPException(401)` in gateway; FastAPI default error responses elsewhere.
- Logging: **no custom logging** in service code (relies on Uvicorn/Celery default stdout, scraped via `kubectl logs`). Alembic has an INI-configured logger.
- Observability: Prometheus metrics on every service, Grafana dashboard, LangFuse traces, GitHub Actions run history. No distributed tracing across services beyond LangFuse's per-LLM-call traces.

### ASCII architecture diagram

```
                          ┌──────────────────────────────────────────────┐
   Developer              │                  AWS (EKS)                     │
   opens/merges PR        │                                                │
        │  webhook        │   ┌─────────┐  verified body   ┌──────────┐    │
        ▼  (HMAC)         │   │ gateway │ ───────────────▶ │ webhook  │    │
   GitHub ───▶ ALB ──────▶│──▶│  :8000  │                  │  :8001   │    │
        ▲                 │   └─────────┘                  └────┬─────┘    │
        │ review posted   │                          write PR │  │ enqueue │
        │                 │                                   ▼  ▼         │
        │                 │                           ┌────────┐ ┌───────┐ │
        │                 │                           │Postgres│ │ Redis │ │
        │                 │                           │ (RDS)  │ │(Celery)│ │
        │                 │                           └───▲────┘ └───┬───┘ │
        │                 │                               │          │     │
        │                 │     ┌──────────────┐  HTTP    │   ┌──────▼────┐│
        │                 │     │ orchestrator │◀─────────┼───│ webhook-  ││
        │                 │     │    :8002     │  analyze  │   │  worker   ││
        │   ┌─────────────┼────▶│  (HPA 2-6)   │           │   └───────────┘│
        │   │ GitHub API  │     │  LangGraph   │ findings  │                │
        │   │ (diff)      │     │  ┌────────┐  │───────────┘                │
        │   │             │     │  │4 agents│  │                            │
        │   │             │     │  │ static │  │   gpt-4o-mini (LangFuse)   │
        │   │             │     │  │ secur. │──┼──────────────▶ OpenAI      │
        │   │             │     │  │ style  │  │                            │
        │   │             │     │  │ arch.  │  │                            │
        │   │             │     │  └───┬────┘  │                            │
        │   │             │     │   merge/dedupe                            │
        │   │             │     └──────┬───────┘                            │
        │   │   ┌─────────┐  post one  │  call reviewer                     │
        └───┼───│reviewer │◀───────────┘                                    │
            │   │  :8003  │ review                                          │
            │   └─────────┘                                                 │
            │                          on merge ─▶ Redis(learning)          │
            │                                 │                             │
            │                          ┌──────▼───────┐  HTTP  ┌─────────┐  │
            │                          │learner-worker│───────▶│ learner │  │
            │                          │ (queue=learn)│        │  :8004  │  │
            │                          └──────────────┘        └────┬────┘  │
            │                                          upsert patterns│      │
            │                                                         ▼      │
            │                                                  ┌────────┐    │
            └──────────────────────────────────────────────── │Postgres│    │
   Observability: Prometheus scrapes /metrics on all 5 svcs;   └────────┘    │
   Grafana dashboards; LangFuse traces; weekly RAGAS Job.                    │
                          └──────────────────────────────────────────────────┘
```

---

## 6. End-to-End Workflows

### Workflow A — Review a newly opened/updated PR (the primary flow)

- **Trigger:** GitHub `pull_request` webhook with `action ∈ {opened, reopened, synchronize}`.
- **Files/functions involved:**
  - `services/gateway/main.py::github_webhook`
  - `services/webhook/main.py::receive_event`, `services/webhook/worker.py::analyze_pr`
  - `services/orchestrator/main.py::analyze`, `get_installation_token`, `fetch_diff`
  - `services/orchestrator/graph.py::build_graph`, `fan_out`, `make_node`, `_style_prompt`, `parse_json_response`, `merge_node`
  - `services/reviewer/main.py::post_review`, `_build_summary`, `get_installation_token`
- **Step-by-step control/data flow:**
  1. GitHub → ALB → **gateway** `POST /webhook/github`. Gateway reads the raw body, recomputes `sha256=HMAC(secret, body)`, constant-time compares to `X-Hub-Signature-256`. On mismatch → `401`. On match → `POST http://webhook:8001/events` with the raw body, `raise_for_status()`, returns `{"status":"ok"}`.
  2. **webhook** `POST /events` parses JSON. For `opened/reopened/synchronize`: extract `pr_number`, `repo_full_name`, `head_sha`, `installation_id`; **dedupe** by selecting an existing `(repo, pr_number, head_sha)` row → if found, return `already_processing`. Otherwise insert a `pull_requests` row (`status="pending"`), commit, refresh, take `pr_id`, then `analyze_pr.apply_async([...], queue="webhook")` and return `202 {"status":"accepted"}`.
  3. **webhook-worker** (Celery, `-Q webhook`) runs `analyze_pr`, which does a synchronous `POST http://orchestrator:8002/analyze` (`timeout=120`).
  4. **orchestrator** `POST /analyze`: `get_installation_token(installation_id)` (RS256 JWT → installation token), `fetch_diff(repo, pr_number, token)` (GitHub `Accept: …v3.diff`). Loads top-10 `patterns` for the repo (ordered by `frequency desc`). Calls `build_graph().invoke({"diff", "patterns", "findings":[]})`.
  5. **LangGraph** `fan_out` (conditional entry) emits four `Send`s → `static_analysis`, `security`, `style`, `architecture` run **in parallel**. Each `make_node`-built node sends `[system prompt, diff]` to `gpt-4o-mini`, parses the JSON array, tags each item with `agent`, returns `{"findings": items}` (reduced via `operator.add`). `style` uses `_style_prompt(state)` which embeds the learned patterns. All four edge into `merge_node`, which de-dupes by `(file,line,agent,message)` and returns `{"findings": merged}` → `END`.
  6. Back in `analyze`, `findings_data = state["findings"]` (⚠️ contains duplicates — see "Side effects/bug"). Each finding is inserted into `findings` (FK `pr_id`). Then `POST http://reviewer:8003/post-review` with the findings (`timeout=60`). Returns `202`.
  7. **reviewer** `POST /post-review`: `get_installation_token` (sync), early-return if no findings. Builds `inline_comments` for findings with a valid `file` and `line>0`, and a markdown `summary`. `POST …/pulls/{n}/reviews` with `event:"COMMENT", body:summary, comments:inline_comments`. If `422` *and* there were inline comments, retry with summary only. `raise_for_status()`. Then `UPDATE pull_requests SET status='reviewed'`. Returns `{"status":"ok"}`.
- **Inputs/outputs:** diff in → posted review + persisted findings out.
- **Side effects:** DB writes (PR row, findings, status); GitHub review; OpenAI spend; LangFuse traces; Prometheus counters.
- **⚠️ Bug in this flow:** because of the `operator.add` reducer + `merge_node` returning under the same `findings` key, the merged list is **appended** to the raw list, so persisted/posted findings are **doubled** (exactly 2× when no two findings are byte-identical).
- **Error behavior:** `401` (bad signature) stops everything; OpenAI/GitHub failures raise inside the Celery task (no retry/back-off in code); `422` → summary-only fallback.
- **Where to look:** `graph.py` (agent logic + the reducer bug), `orchestrator/main.py` (token/diff/persist), `reviewer/main.py` (posting + fallback).

### Workflow B — Learn from a merged PR

- **Trigger:** `pull_request` webhook with `action == "closed"` and `pull_request.merged == true`.
- **Files/functions:** `services/webhook/main.py::receive_event` (merge branch), `services/webhook/worker.py::trigger_learning`, `services/learner/worker.py` (Celery app/route), `services/learner/main.py::learn`.
- **Flow:**
  1. webhook `receive_event` matches the merge branch, looks up the PR by `(repo, pr_number)` via `scalar_one_or_none()`, and if found `trigger_learning.apply_async([repo, str(pr.id)], queue="learning")`; returns `accepted`.
  2. **Intended:** `learner-worker` (`-Q learning`) runs `trigger_learning`, which `POST http://learner:8004/learn`.
  3. **learner** `POST /learn` selects this PR's `findings` with severity in `{warning, error}`, and for each does an **upsert** into `patterns` (`INSERT … ON CONFLICT (repo_full_name, pattern_text) DO UPDATE SET frequency = frequency + 1`). Commits.
- **Outputs:** new/updated `patterns` rows; these later feed Workflow A's style agent.
- **⚠️ Two latent bugs visible from code:**
  1. **Celery task not registered in the consumer.** `trigger_learning`'s function body lives in `services/webhook/worker.py`, but the `learner-worker` deployment runs `celery -A worker worker -Q learning` against `services/learner/worker.py`, which only declares the Celery app and a route — **no `@app.task(name="trigger_learning")`**. A worker consuming a task name it hasn't registered raises Celery `NotRegistered`. So, *as wired in the repo*, the queued `trigger_learning` message appears unable to execute on the learner-worker, and the `/learn` HTTP call would never be made through the queue. (Stated as a code-evidence-based concern; confirming requires running the deployed workers. See §17 for the small fix.)
  2. **`scalar_one_or_none()` on a non-unique key.** The lookup filters only `(repo_full_name, pr_number)` — but `synchronize` events create *multiple* rows per PR (different `head_sha`). If ≥2 rows exist, `scalar_one_or_none()` raises `MultipleResultsFound`, breaking the merge handler for any PR that received commits after opening.
- **Where to look:** `webhook/main.py` (merge branch + lookup), `webhook/worker.py` vs `learner/worker.py` (the registration gap), `learner/main.py` (upsert).

### Workflow C — Weekly offline evaluation (RAGAS)

- **Trigger:** GitHub Actions `Evaluate` workflow — cron `0 9 * * 1` (Mondays 09:00 UTC) or manual `workflow_dispatch`.
- **Files:** `.github/workflows/evaluate.yml`, `scripts/Dockerfile`, `scripts/evaluate.py`, `infra/k8s/evaluate-job.yaml`.
- **Flow:** Actions assumes the AWS role (OIDC) → builds & pushes the `evaluate` image to ECR → `aws eks update-kubeconfig` → deletes any old `evaluate` Job → `sed` substitutes `IMAGE_PLACEHOLDER` in `evaluate-job.yaml` and `kubectl apply`s it → waits for the Job to complete/fail → prints logs → cleans up the Job. Inside the Job, `evaluate.py` connects to Postgres (psycopg2, after rewriting the URL to drop `+asyncpg`), pulls the last 50 findings joined to PRs, builds a RAGAS `Dataset` (question = "What issues exist in this code?", answer = finding message, contexts = `[file]`), runs `faithfulness` + `answer_relevancy`, prints scores, and **exits `1` if mean faithfulness < 0.7**.
- **Outputs:** pass/fail Actions run + printed scores. No DB writes.
- **Caveat:** the "context" is only the **filename**, not the actual diff/code, so faithfulness scoring is weakly grounded; and a single test PR yields too little data to be meaningful (acknowledged in README).

### Workflow D — Deploy a service (CI/CD)

- **Trigger:** push to `main` touching `services/<svc>/**` (path filter) — including bumping `services/<svc>/deploy.txt`.
- **Files:** `.github/workflows/<svc>.yml` → `.github/workflows/service-ci.yml` (reusable).
- **Flow:** `test` (setup Python 3.11, `pip install -r requirements.txt`, `pytest` iff `tests/` exists) → `build-and-push` (OIDC, ECR login, `docker build` tagged with `$GITHUB_SHA` and `latest`, push both) → `deploy` (OIDC, `aws eks update-kubeconfig`, `kubectl set image deployment/$SERVICE …:$GITHUB_SHA`). The build/deploy jobs run only on `push` to `main`.
- **Outputs:** new image in ECR + rolling update of one Deployment.
- **Note:** the worker Deployments (`webhook-worker`, `learner-worker`) and the `*-lb` Service are **not** updated by `kubectl set image deployment/$SERVICE` — only the same-named Deployment is. Workers pick up new images on the next manual `kubectl rollout restart` / re-apply, or when their pods are recreated pulling `:latest`. *(Behavior inferred from the manifests + pipeline; not separately documented in the repo.)*

### Workflow E — Provision / tear down infrastructure

- **Trigger:** operator runs `terraform apply` / `terraform destroy` locally.
- **Files:** `infra/terraform/{main,variables,outputs}.tf`, plus the `INSTRUCTIONS.md` cleanup ordering (delete k8s ingress/services + leftover SGs/LBs *before* `terraform destroy` to avoid `DependencyViolation`).
- **Outputs:** the full AWS stack created/destroyed; `terraform output` yields cluster/RDS/Redis/ECR endpoints used to fill the ConfigMap/Secret. README reports `terraform destroy` removed **66 resources** to reach $0.

---

## 7. Data Model, State, And Configuration

### 7.1 Database schema (Postgres; created by `db/migrations/versions/0001_initial.py`)

Extension: `pgcrypto` (for `gen_random_uuid()`).

**`pull_requests`**

| Column | Type | Constraints / default | Meaning |
|---|---|---|---|
| `id` | UUID | PK, `gen_random_uuid()` | PR analysis record id (used as `pr_id` FK target). |
| `repo_full_name` | Text | NOT NULL | e.g. `owner/repo`. |
| `pr_number` | Integer | NOT NULL | PR number. |
| `head_sha` | Text | NOT NULL | Head commit SHA (part of app-level dedupe key). |
| `installation_id` | BigInteger | NOT NULL | GitHub App installation id. |
| `status` | Text | NOT NULL default `pending` | `pending` → `reviewed`. |
| `created_at` | timestamptz | default `now()` | Insert time. |

> ⚠️ **No DB-level unique constraint** on `(repo_full_name, pr_number, head_sha)`. Dedup is best-effort at the application layer (SELECT-then-INSERT), so concurrent identical webhooks can race.

**`findings`**

| Column | Type | Constraints | Meaning |
|---|---|---|---|
| `id` | UUID | PK | Finding id. |
| `pr_id` | UUID | FK → `pull_requests.id`, nullable | Owning PR. |
| `file` | Text | nullable | File path from the model. |
| `line` | Integer | nullable | Line number from the model (often unreliable). |
| `severity` | Text | nullable | `info` / `warning` / `error`. |
| `message` | Text | nullable | The finding text. |
| `agent` | Text | nullable | `static_analysis` / `security` / `style` / `architecture`. |
| `created_at` | timestamptz | default `now()` | Insert time. |

**`patterns`**

| Column | Type | Constraints | Meaning |
|---|---|---|---|
| `id` | UUID | PK | Pattern id. |
| `repo_full_name` | Text | NOT NULL | Repo the pattern belongs to. |
| `pattern_text` | Text | NOT NULL | The recurring finding message. |
| `frequency` | Integer | default `1` | Times seen (incremented on conflict). |
| `updated_at` | timestamptz | default `now()` | Last upsert time. |
| — | — | **UNIQUE (`repo_full_name`, `pattern_text`)** | Enables `ON CONFLICT` upsert. |

> **Model duplication:** the SQLAlchemy ORM models are **redefined per service** in each `models.py` (no shared package). They are consistent, with one cosmetic difference: webhook/reviewer declare `status` as `NOT NULL`, orchestrator omits `nullable=False` on `status`. The migration is the source of truth (`status` NOT NULL). The webhook image also bundles `db/` for the migration job; other services do not run migrations.

### 7.2 LangGraph state (`services/orchestrator/graph.py`)

```python
class GraphState(TypedDict):
    diff: str                                   # the unified PR diff
    patterns: list[str]                         # top-10 learned patterns for the repo
    findings: Annotated[list[dict], operator.add]   # reducer = list concatenation
```

- The `Annotated[..., operator.add]` **reducer** is the crux of the double-emit bug: every node's returned `findings` is *concatenated* onto the accumulated list, including `merge_node`'s output.
- Each finding dict: `{"file": str|None, "line": int|None, "severity": str, "message": str, "agent": str}` (the `agent` key is injected by `make_node`).

### 7.3 API request/response shapes (Pydantic models)

| Model | File | Fields |
|---|---|---|
| `AnalyzeRequest` | orchestrator/models.py | `pr_id: UUID, pr_number: int, repo_full_name: str, head_sha: str, installation_id: int` |
| `ReviewRequest` | reviewer/models.py | `pr_id: UUID, repo_full_name: str, pr_number: int, installation_id: int, findings: list[dict[str, Any]]` |
| `LearnRequest` | learner/models.py | `repo_full_name: str, pr_id: UUID` |

Loose JSON shapes (not Pydantic-validated): the GitHub webhook event consumed by `webhook/main.py` (`action`, `pull_request.{number,merged,head.sha}`, `repository.full_name`, `installation.id`), and the GitHub review payload `{event, body, comments[]}` sent by the reviewer.

### 7.4 Configuration objects (`Settings(BaseSettings)`, `pydantic-settings`)

Each service has its own `Settings` with `env_file = ".env"` and these fields (defaults shown where present):

| Service | Settings fields |
|---|---|
| gateway | `github_webhook_secret=""` |
| webhook | `database_url="postgresql+asyncpg://user:password@postgres:5432/codereviewer"`, `redis_url="redis://redis:6379/0"` |
| orchestrator | the two above + `github_app_id`, `github_app_private_key`, `openai_api_key`, `langfuse_public_key`, `langfuse_secret_key`, `langfuse_host="http://langfuse:3000"` |
| reviewer | `database_url`, `github_app_id`, `github_app_private_key` |
| learner | `database_url`, `redis_url` |

> Env-var mapping is case-insensitive in pydantic-settings, so `DATABASE_URL` (k8s) → `database_url` (field). The orchestrator's `openai_api_key` is declared but the OpenAI/LangFuse clients also read `OPENAI_API_KEY` / `LANGFUSE_*` straight from the environment.

### 7.5 Constants & validation rules

- **Agent prompts** (`PROMPTS` dict + `_style_prompt`) — the only "business rules" of the review; each instructs the model to return a JSON array of `{file,line,severity,message}`.
- **JWT timing:** `iat = now-60`, `exp = now+600` (10-minute app JWT; the 60s back-date absorbs clock skew).
- **Severity allow-list for learning:** only `{"warning","error"}` findings become patterns.
- **Inline-comment rule (reviewer):** include only findings with truthy `file` and integer `line > 0`; `side="RIGHT"`.
- **Eval gate:** mean faithfulness `< 0.7` → fail.
- **HPA:** orchestrator CPU target 70%, replicas 2–6.

### 7.6 File formats

- YAML (k8s, Actions, Prometheus), HCL (Terraform), JSON (Grafana dashboard), INI (`alembic.ini`), `requirements.txt`, Dockerfile, Markdown, a binary **PDF** (`ai-code-review-pr-screenshot.pdf`), and `deploy.txt` (a one-line redeploy trigger containing `8`, with a UTF-8 BOM).

### 7.7 Feature flags

None. There is no feature-flag system; behavior is governed by env vars and code.

---

## 8. API, Routes, Commands, And Entrypoints

### 8.1 HTTP endpoints

| Method & Path | Service | Called by | Calls | Result |
|---|---|---|---|---|
| `POST /webhook/github` | gateway `:8000` | GitHub (via ALB) | `webhook:8001/events` | `{"status":"ok"}` or `401`. |
| `GET /health` | all 5 | k8s liveness/readiness probes, ALB health check | — | `{"status":"ok"}`. |
| `GET /metrics` | all 5 | Prometheus | — | Prometheus exposition (added by `Instrumentator`). |
| `POST /events` (`202`) | webhook `:8001` | gateway | DB; Celery enqueue | `accepted` / `skipped` / `already_processing`. |
| `POST /analyze` (`202`) | orchestrator `:8002` | webhook-worker | GitHub API, OpenAI, DB; `reviewer:8003/post-review` | `accepted`. |
| `POST /post-review` | reviewer `:8003` | orchestrator | GitHub API, DB | `{"status":"ok"}`. |
| `POST /learn` | learner `:8004` | learner-worker (intended) | DB upsert | `{"status":"ok"}`. |

### 8.2 Celery tasks (queues)

| Task | Defined in | Queue | Consumer | Action |
|---|---|---|---|---|
| `analyze_pr` | webhook/worker.py | `webhook` | webhook-worker | `POST orchestrator:8002/analyze`. |
| `trigger_learning` | webhook/worker.py | `learning` | learner-worker (but task **not registered** there — see §6/§17) | `POST learner:8004/learn`. |

### 8.3 Server entrypoints / bootstrap

- `uvicorn main:app --host 0.0.0.0 --port {8000..8004}` — one per service (Dockerfile `CMD`).
- Each `main.py` constructs `app = FastAPI()` and immediately calls `Instrumentator().instrument(app).expose(app)`; the four DB-backed services also create the async engine + `AsyncSessionLocal` at import time.

### 8.4 Background-job entrypoints

- `celery -A worker worker --loglevel=info -Q webhook` (webhook-worker).
- `celery -A worker worker --loglevel=info -Q learning` (learner-worker).
- `alembic -c /db/alembic.ini upgrade head` (db-migrate Job).
- `python evaluate.py` (evaluate Job).

### 8.5 Scripts & package exports

- `scripts/evaluate.py::main()` — the only standalone script (`if __name__ == "__main__": main()`).
- No installable package / `setup.py` / `pyproject.toml`; nothing is exported as a library.

### 8.6 Event handlers

- The single inbound event is the GitHub `pull_request` webhook, branched in `webhook/main.py::receive_event` by `action` (+ `merged`).

---

## 9. Full Repository Map

```
github-PR-reviewer-AI-Agent/
├── .gitignore                         # ignore rules; protects .env/secrets/tfstate/secret.local.yaml
├── INSTRUCTIONS.md                    # 1,881-line, 16-phase production runbook (Windows CMD-oriented)
├── LICENSE                            # MIT, © 2026 Srikara S
├── README.md                          # project overview, architecture (mermaid), decisions, results, limits
├── ai-code-review-pr-screenshot.pdf   # binary (955 KB): printout of a real review run (proof artifact)
│
├── .github/workflows/                 # CI/CD
│   ├── service-ci.yml                 # REUSABLE pipeline: test → build/push (ECR) → deploy (kubectl set image)
│   ├── gateway.yml                    # path-filtered trigger → calls service-ci with service=gateway
│   ├── webhook.yml                    # …service=webhook
│   ├── orchestrator.yml               # …service=orchestrator
│   ├── reviewer.yml                   # …service=reviewer
│   ├── learner.yml                    # …service=learner
│   └── evaluate.yml                   # weekly cron + manual: build evaluate image, run RAGAS k8s Job
│
├── db/                                # database migrations
│   ├── alembic.ini                    # Alembic config (sync default URL; overridden by DATABASE_URL)
│   └── migrations/
│       ├── env.py                     # async Alembic runner (asyncpg engine, NullPool)
│       └── versions/
│           └── 0001_initial.py        # creates pgcrypto + pull_requests/findings/patterns tables
│
├── infra/
│   ├── terraform/                     # AWS Infrastructure-as-Code
│   │   ├── main.tf                    # VPC, EKS, RDS, ElastiCache, IAM (LBC), S3, 6 ECR repos, SGs
│   │   ├── variables.tf               # region, cluster_name, db_password (sensitive), environment
│   │   └── outputs.tf                 # EKS/RDS/Redis endpoints + 5 ECR URLs
│   └── k8s/                           # Kubernetes manifests
│       ├── configmap.yaml             # non-secret env (LANGFUSE_HOST, REDIS_URL, service URLs)
│       ├── secret.yaml                # PLACEHOLDER Secret (empty values; real ones via local overlay)
│       ├── gateway.yaml               # Deployment + ClusterIP + LoadBalancer Service (:8000)
│       ├── webhook.yaml               # Deployment + ClusterIP (:8001)
│       ├── webhook-worker.yaml        # Celery worker Deployment (-Q webhook)
│       ├── orchestrator.yaml          # Deployment + ClusterIP (:8002)
│       ├── reviewer.yaml              # Deployment + ClusterIP (:8003)
│       ├── learner.yaml               # Deployment + ClusterIP (:8004)
│       ├── learner-worker.yaml        # Celery worker Deployment (-Q learning)
│       ├── hpa.yaml                   # HorizontalPodAutoscaler for orchestrator (2–6, 70% CPU)
│       ├── ingress.yaml               # ALB ingress → gateway:8000
│       ├── migration-job.yaml         # Job: alembic upgrade head (webhook image)
│       ├── evaluate-job.yaml          # Job: RAGAS evaluation (IMAGE_PLACEHOLDER)
│       └── kubectl                     # 0-byte stray file (accidentally committed; not a binary)
│
├── monitoring/
│   ├── prometheus.yml                 # scrape config for the 5 services' /metrics
│   └── grafana-dashboard.json         # importable dashboard: per-service rate/latency/errors + health row
│
├── scripts/
│   ├── Dockerfile                     # evaluate image (pinned ragas/langchain-openai + psycopg2/datasets)
│   └── evaluate.py                    # RAGAS faithfulness/answer-relevancy over last 50 findings
│
└── services/                          # the application code (5 services; 2 also ship a worker)
    ├── gateway/        {Dockerfile, deploy.txt, main.py, models.py, requirements.txt}
    ├── webhook/        {Dockerfile, deploy.txt, main.py, models.py, requirements.txt, worker.py}
    ├── orchestrator/   {Dockerfile, deploy.txt, graph.py, main.py, models.py, requirements.txt}
    ├── reviewer/       {Dockerfile, deploy.txt, main.py, models.py, requirements.txt}
    └── learner/        {Dockerfile, deploy.txt, main.py, models.py, requirements.txt, worker.py}
```

**One-line purpose per tracked file** (deep detail in §10):

| Path | Purpose |
|---|---|
| `.gitignore` | Exclude env/secrets, Python/Terraform/k8s artifacts; allow-list `.env.example`. |
| `INSTRUCTIONS.md` | End-to-end setup/operate/teardown runbook (16 phases) for Windows. |
| `LICENSE` | MIT license. |
| `README.md` | The canonical narrative: what/why/how/decisions/results/limitations. |
| `ai-code-review-pr-screenshot.pdf` | Binary proof of a real review run. |
| `.github/workflows/service-ci.yml` | Reusable 3-stage per-service pipeline. |
| `.github/workflows/{gateway,webhook,orchestrator,reviewer,learner}.yml` | Path-filtered triggers wiring each service to the reusable pipeline. |
| `.github/workflows/evaluate.yml` | Weekly + manual RAGAS evaluation pipeline. |
| `db/alembic.ini` | Alembic configuration. |
| `db/migrations/env.py` | Async migration runner. |
| `db/migrations/versions/0001_initial.py` | Initial schema (3 tables + pgcrypto). |
| `infra/terraform/main.tf` | All AWS resources. |
| `infra/terraform/variables.tf` | Terraform inputs. |
| `infra/terraform/outputs.tf` | Terraform outputs (endpoints/URLs). |
| `infra/k8s/configmap.yaml` | Non-secret runtime config. |
| `infra/k8s/secret.yaml` | Placeholder Secret schema. |
| `infra/k8s/gateway.yaml` … `learner.yaml` | Per-service Deployments + Services. |
| `infra/k8s/webhook-worker.yaml`, `learner-worker.yaml` | Celery worker Deployments. |
| `infra/k8s/hpa.yaml` | Orchestrator autoscaler. |
| `infra/k8s/ingress.yaml` | ALB ingress. |
| `infra/k8s/migration-job.yaml` | DB migration Job. |
| `infra/k8s/evaluate-job.yaml` | Evaluation Job template. |
| `infra/k8s/kubectl` | Empty stray file. |
| `monitoring/prometheus.yml` | Prometheus scrape targets. |
| `monitoring/grafana-dashboard.json` | Grafana dashboard definition. |
| `scripts/Dockerfile` | Evaluation image build. |
| `scripts/evaluate.py` | Offline RAGAS evaluation. |
| `services/*/Dockerfile` | Per-service image builds. |
| `services/*/deploy.txt` | One-line redeploy triggers (`8`). |
| `services/*/main.py` | FastAPI apps / endpoints. |
| `services/*/models.py` | ORM models + Pydantic `Settings`/requests. |
| `services/*/requirements.txt` | Per-service deps. |
| `services/orchestrator/graph.py` | The LangGraph 4-agent graph. |
| `services/webhook/worker.py`, `services/learner/worker.py` | Celery apps/tasks. |

---

## 10. File-By-File Deep Dive

> Every tracked file is covered. Highly repetitive files (the six Dockerfiles, five `deploy.txt`, five near-identical service Deployments, five near-identical per-service workflow triggers) are documented once in full, then their per-file deltas are listed. Binary/empty/generated files are documented at the appropriate level with a note on why line-by-line analysis doesn't apply.

### 10.1 Application services (`services/`)

#### `services/gateway/main.py`

**Role:** The security edge. Verifies that an incoming webhook genuinely came from GitHub, then forwards the raw body to the `webhook` service. It holds **no business logic** by design.

**Why it matters:** Cleanly separates "is this really from GitHub?" from everything downstream. If verification fails, nothing else in the system runs.

**Key dependencies/imports:** `hashlib`, `hmac` (signature math); `httpx` (async forward); `fastapi` (`FastAPI/HTTPException/Request`); `prometheus_fastapi_instrumentator` (`/metrics`); `models.Settings` (the webhook secret).

**Exports/public surface:** `app` (ASGI), `health()`, `github_webhook()`.

**Used by:** GitHub (via ALB → ingress → `gateway` Service). Forwards to `webhook:8001/events`.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| `settings = Settings()` / `app` / `Instrumentator…` (L11–13) | Load secret; create app; expose `/metrics`. | env `GITHUB_WEBHOOK_SECRET` | running app | binds metrics route | If the secret is empty (default `""`), HMAC will never match a real GitHub signature → all `401`. |
| `health()` (L16–18) | Liveness/readiness probe target. | — | `{"status":"ok"}` | none | Used by k8s probes + ALB health check. |
| `github_webhook()` (L21–46) | Read raw body, recompute `sha256=HMAC(secret, body)`, constant-time compare to `X-Hub-Signature-256`; on match, forward raw body to webhook and `raise_for_status()`. | raw request body + signature header | `{"status":"ok"}` or `401` | outbound POST to webhook | Uses `hmac.compare_digest` (timing-safe). Must hash the **raw** bytes (it does — `await request.body()`), since re-serializing JSON would change the signature. If webhook returns ≥400, `raise_for_status()` makes gateway 500 to GitHub (triggers GitHub redelivery). |

**Interview talking points:** Why a dedicated verification edge; why raw-body hashing matters; constant-time comparison to avoid timing attacks; the App-vs-PAT distinction.

**Risks/improvements:** No timeout on the forward `httpx.AsyncClient()` call (could hang a worker thread); a new client per request (no connection pooling); surfacing webhook 5xx as gateway 5xx is reasonable (lets GitHub retry) but undifferentiated.

#### `services/gateway/models.py`

**Role:** Defines `Settings(BaseSettings)` with the single field `github_webhook_secret`.
**Why it matters:** Centralizes the one secret the gateway needs; `pydantic-settings` reads it from env (or `.env`).
**Public surface:** `Settings`.
**Walkthrough:** Trivial — one field defaulting to `""`, `Config.env_file=".env"`. **Edge case:** the empty default means a misconfigured deploy silently rejects everything with `401` rather than failing fast at startup.

#### `services/webhook/main.py`

**Role:** Ingest the verified GitHub event; persist the PR; enqueue analysis; handle merges by enqueuing learning. The asynchronous "front door" to the pipeline.

**Why it matters:** Implements the **decoupling** (return `202` fast, do work later) and the **idempotency** (dedupe) that make webhook handling safe.

**Key dependencies/imports:** `fastapi`; `sqlalchemy.select` + async engine/session; `models` (`Settings, PullRequest, Base`); `worker` (`analyze_pr, trigger_learning`); Prometheus instrumentator.

**Exports/public surface:** `app`, `health()`, `receive_event()`, plus module-level `engine`/`AsyncSessionLocal`.

**Used by:** gateway (`/events`). Enqueues to Redis; rows read later by orchestrator/learner.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| L10–15 | Create async engine + session factory (`expire_on_commit=False`); app + metrics. | `DATABASE_URL` | app | DB engine | Engine created at import; bad URL → crash loop. |
| `receive_event` merge branch (L29–42) | If `action=="closed"` and `pull_request.merged`, look up PR by `(repo, pr_number)` and enqueue `trigger_learning` to queue `learning`; return `accepted`. | event JSON | `accepted` | Celery enqueue | ⚠️ `scalar_one_or_none()` raises if >1 row (multiple `head_sha`s); silently does nothing if PR was never recorded. |
| action filter (L44–45) | Ignore actions other than `opened/reopened/synchronize`. | `action` | `skipped` | none | Webhooks for labels/comments/etc. are dropped early. |
| dedupe (L52–61) | SELECT by `(repo, pr_number, head_sha)`; if exists → `already_processing`. | event fields | `already_processing` | DB read | App-level only (no unique index) → race window between SELECT and INSERT. |
| insert + enqueue (L63–76) | Insert `pending` PR row, commit, refresh, enqueue `analyze_pr` to queue `webhook`; `202 accepted`. | event fields | `accepted` | DB write + Celery enqueue | `installation_id` defaults to `0` if absent (would later fail token mint). |

**Interview talking points:** 202-and-enqueue pattern; idempotency key choice; why `expire_on_commit=False`; the merge vs open branching.

**Risks/improvements:** the `scalar_one_or_none()` merge bug; missing DB unique constraint; no validation of the event shape (KeyErrors avoided via `.get` defaults, but malformed events still enqueue with empty strings).

#### `services/webhook/models.py`

**Role:** `Base`, the `PullRequest` ORM model, and `Settings` (`database_url`, `redis_url`).
**Why it matters:** Declares the table the webhook writes and the connection settings.
**Walkthrough:** `PullRequest` matches the migration (here `status` is `NOT NULL default "pending"`). `Settings` defaults point at in-cluster hostnames (`postgres`, `redis`). **Note:** model duplicated across services (no shared lib).

#### `services/webhook/worker.py`

**Role:** The Celery app + the two task **bodies** (`analyze_pr`, `trigger_learning`).

**Why it matters:** This is where queued work actually calls the next service. `analyze_pr` → orchestrator; `trigger_learning` → learner.

**Key imports:** `httpx` (sync `Client`), `celery.Celery`, `os`.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| L5–10 | Create `Celery("webhook", broker/backend=REDIS_URL)`; route `analyze_pr→webhook`, `trigger_learning→learning`. | `REDIS_URL` env | Celery app | connects to Redis | Reads `REDIS_URL` directly from env (not via `Settings`). |
| `analyze_pr` (L13–26) | Sync `POST orchestrator:8002/analyze` with the PR identity (`timeout=120`). | task args | none | HTTP call | No retry/error handling; a non-2xx isn't raised (no `raise_for_status`), so failures are swallowed silently. |
| `trigger_learning` (L29–36) | Sync `POST learner:8004/learn` (`timeout=60`). | repo, pr_id | none | HTTP call | ⚠️ Lives **here**, but the learner-worker (which consumes `learning`) doesn't register it (see §6/§17). |

**Interview talking points:** task routing by queue; why workers use the **sync** httpx client; the deliberate `timeout` to bound a hung call.

**Risks/improvements:** no `raise_for_status()` → silent failures; no idempotency/retry; the cross-image task-registration gap.

#### `services/orchestrator/main.py`

**Role:** The brain's coordinator: mint a GitHub token, fetch the diff, load learned patterns, run the LangGraph graph, persist findings, and hand off to the reviewer.

**Why it matters:** Ties GitHub auth + data loading + the agent graph + persistence + downstream call into one request.

**Key imports:** `time`, `httpx`, `jwt` (PyJWT), `fastapi`, async SQLAlchemy, `models` (`Settings, Finding, Pattern, AnalyzeRequest`), `graph.build_graph`.

**Exports:** `app`, `health()`, `analyze()`, `get_installation_token()`, `fetch_diff()`.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| `analyze` token+diff (L28–30) | `await get_installation_token`; `await fetch_diff`. | `AnalyzeRequest` | token, diff | GitHub API calls | Both `raise_for_status()`; failure aborts the request. |
| load patterns (L32–39) | SELECT top-10 patterns for repo by `frequency desc`. | repo | `list[str]` | DB read | Empty list if no patterns yet. |
| run graph (L41–42) | **Synchronous** `build_graph().invoke({...})`. | diff, patterns | `state.findings` | OpenAI calls | ⚠️ Blocks the event loop (sync call in async handler) for the full multi-agent run; ⚠️ findings are doubled by the reducer bug. Compiles the graph **per request** (small overhead). |
| persist findings (L44–54) | Insert each finding (`pr_id` + fields). | findings | DB rows | DB write | Persists the duplicated findings too. |
| call reviewer (L56–67) | `POST reviewer:8003/post-review` with findings (`timeout=60`). | findings | — | HTTP call | No `raise_for_status` on this one → reviewer failure is swallowed; returns `202` regardless. |
| `get_installation_token` (L72–87) | Build RS256 JWT (`iat-60/exp+600/iss=app_id`), un-escape `\n` in key, exchange for installation token. | installation_id | token str | GitHub API call | `raise_for_status`; relies on correct PEM formatting. |
| `fetch_diff` (L90–100) | GET the PR with `Accept: …v3.diff`. | repo, pr#, token | unified diff text | GitHub API call | Returns `.text`; large diffs are sent verbatim to the model (no truncation/token budgeting). |

**Interview talking points:** GitHub App JWT→installation-token exchange; conditioning the style agent on learned patterns; the sync-graph-in-async-handler tradeoff; per-request graph compilation.

**Risks/improvements:** event-loop blocking; missing `raise_for_status` on the reviewer call; no diff size/token guard; the double-emit bug originates in the graph but is *persisted* here.

#### `services/orchestrator/graph.py`  ⭐ (core AI logic)

**Role:** Defines the LangGraph multi-agent graph: prompts, JSON parsing, per-agent nodes, the style prompt builder, the merge/dedupe node, the fan-out entry, and graph assembly.

**Why it matters:** This *is* the "AI" of the product — the multi-agent design the project is built to showcase.

**Key imports:** `json, operator, re`; `typing` (`TypedDict, Annotated`); `langfuse.openai.OpenAI` (traced drop-in); `langgraph.graph` (`StateGraph, END`); `langgraph.constants.Send`.

**Exports:** `PROMPTS`, `parse_json_response`, `GraphState`, `make_node`, `_style_prompt`, `merge_node`, `fan_out`, `build_graph`.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| `client = OpenAI()` (L10) | Instantiate the **LangFuse-wrapped** OpenAI client at import. | env keys | client | enables tracing | Module-level singleton shared by all nodes. |
| `PROMPTS` (L12–16) | System prompts for static_analysis / security / architecture (style is dynamic). | — | dict | none | Each demands "only a JSON array" of `{file,line,severity,message}`. |
| `parse_json_response` (L19–27) | Strip, pull JSON out of a ```` ```json ```` fence if present, `json.loads`; on any error return `[]`. | raw model text | `list` | none | Robust to fenced output; **silently** returns `[]` on malformed JSON (a finding could be dropped without trace). |
| `GraphState` (L30–33) | Typed state; `findings` has an `operator.add` reducer. | — | type | none | ⚠️ The reducer is the root of the double-emit bug. |
| `make_node` (L36–50) | Factory: build a node that resolves its prompt (static or callable), calls `gpt-4o-mini` with `[system, user=diff]`, parses items, tags each with `agent`, returns `{"findings": items}`. | agent name, prompt-or-fn | node fn | OpenAI call | No temperature/max_tokens set (SDK defaults); no per-call error handling beyond the parser's `[]`. |
| `_style_prompt` (L53–55) | Build the style system prompt, embedding the repo's patterns (or "None"). | state | prompt str | none | This is the **learning feedback loop's** read side. |
| `merge_node` (L58–66) | De-dupe findings by `(file,line,agent,message)`; return `{"findings": merged}`. | state.findings | deduped list | none | ⚠️ Returns under `findings`, so the reducer **appends** instead of replacing → duplicates. |
| `fan_out` (L69–75) | Conditional entry: emit four `Send(node, state)` → parallel execution. | state | list[Send] | none | Each agent gets the full state (incl. diff + patterns). |
| `build_graph` (L78–93) | Add 4 agent nodes + merge; `set_conditional_entry_point(fan_out)`; edge each agent→merge; merge→END; `compile()`. | — | compiled graph | none | Compiled fresh on each `build_graph()` call (orchestrator calls it per request). |

**Interview talking points:** the `Send` fan-out for parallelism; factory pattern for agents; static vs dynamic (callable) prompts; LangFuse drop-in for zero-touch tracing; **and** the ability to explain the reducer bug precisely and its one-line fix.

**Possible improvements/risks:** fix the double-emit (write merged to a separate key, or dedupe at save time); ground line numbers against the diff; add structured-output/function-calling to harden JSON; set `temperature=0` for determinism; surface parse failures instead of silent `[]`; cap diff size; compile the graph once at module load.

#### `services/orchestrator/models.py`

**Role:** ORM models (`PullRequest`, `Finding`, `Pattern`), the `AnalyzeRequest` Pydantic model, and the richest `Settings` (DB, Redis, GitHub App, OpenAI, LangFuse).
**Why it matters:** Declares everything the orchestrator reads/writes + all its secrets.
**Walkthrough:** Models mirror the migration (note: `status` lacks `nullable=False` here — cosmetic). `AnalyzeRequest` enforces types on the inbound payload. `Settings` has the full key set; `langfuse_host` defaults to a local dev value overridden by the ConfigMap.

#### `services/reviewer/main.py`

**Role:** Turn findings into a single GitHub review (summary + inline comments), with a `422`→summary fallback, then mark the PR `reviewed`.

**Why it matters:** The user-visible output of the whole system lands here.

**Key imports:** `time, httpx, jwt`, fastapi, async SQLAlchemy (`update`), `models` (`Settings, PullRequest, ReviewRequest`).

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| `_finding_summary_line` (L26–28) | Format one finding as a markdown block (`[SEVERITY] file:line (agent) message`). | finding dict | str | none | Defaults: severity `info`, file `unknown`, line `?`. |
| `_build_summary` (L31–33) | Join a header + all finding lines. | findings | markdown | none | Posts **all** findings (incl. duplicates from the bug). |
| `post_review` token + guard (L37–41) | Mint token (sync); return early if no findings. | `ReviewRequest` | maybe `{"status":"ok"}` | GitHub call | Early exit avoids empty reviews. |
| build inline comments (L43–55) | For findings with `file` and integer `line>0`, build `{path,line,side:"RIGHT",body}`. | findings | list | none | Coerces `line` via `int(... or 0)`; bad values → skipped. |
| post + fallback (L64–78) | POST review with comments; if `422` and comments existed, retry summary-only; `raise_for_status`. | payload | GitHub review | **posts review** | The fallback is the documented "inline comments often become a summary" behavior. |
| mark reviewed (L80–84) | `UPDATE pull_requests SET status='reviewed'`. | pr_id | — | DB write | Runs only after a successful post. |
| `get_installation_token` (L89–104) | Same JWT exchange as orchestrator, but **synchronous** (`httpx.Client`). | installation_id | token | GitHub call | Sync call inside an async endpoint → blocks the loop briefly. |

**Interview talking points:** one-review-per-PR UX; the `422` fallback as a pragmatic robustness choice; severity/agent surfaced in each comment.

**Risks/improvements:** ground line numbers (root cause of fallbacks); make token mint async; dedupe before posting (mitigates the double-emit user-visibly).

#### `services/reviewer/models.py`

**Role:** `PullRequest` ORM (for the status update), `ReviewRequest` Pydantic model, and a slim `Settings` (DB + GitHub App only — no Redis/OpenAI).
**Walkthrough:** `ReviewRequest.findings: list[dict[str, Any]]` deliberately accepts loose dicts (the findings are model-generated). `Settings` shows the reviewer needs only DB + GitHub creds.

#### `services/learner/main.py`

**Role:** On merge, read a PR's `warning`/`error` findings and upsert them into per-repo `patterns` with a frequency counter.

**Why it matters:** The write side of the learning feedback loop.

**Key imports:** fastapi; `sqlalchemy.select/update`; `sqlalchemy.dialects.postgresql.insert` (for `ON CONFLICT`); async session; `models` (`Settings, Finding, Pattern, LearnRequest`).

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| `learn` select (L27–34) | SELECT findings for `pr_id` where severity ∈ {warning,error}. | `LearnRequest` | findings | DB read | `info` findings are ignored by design. |
| upsert loop (L36–49) | For each, `INSERT … ON CONFLICT (repo_full_name, pattern_text) DO UPDATE SET frequency = frequency+1`. | findings | — | DB upsert | Relies on the UNIQUE constraint from the migration. Uses PG-specific `insert`. |
| commit (L51) | Commit the batch. | — | `{"status":"ok"}` | DB commit | All-or-nothing per request. |

**Interview talking points:** Postgres `ON CONFLICT` upsert as a frequency counter; severity gating; why this is "learning without fine-tuning."

**Risks/improvements:** the duplicated findings inflate frequencies faster than reality; no normalization of `pattern_text` (near-duplicate messages won't merge); reachability depends on fixing the Celery registration gap (else `/learn` is only reachable by a direct HTTP call, never via the queue).

#### `services/learner/models.py`

**Role:** `Finding` + `Pattern` ORM, `LearnRequest`, and `Settings` (DB + Redis).
**Walkthrough:** Note it defines `Finding`/`Pattern` but **not** `PullRequest` (it doesn't need it). Otherwise mirrors the shared schema.

#### `services/learner/worker.py`

**Role:** The learner's Celery app + route map for the `learning` queue.
**Why it matters:** This is the consumer the `learner-worker` Deployment runs.
**Walkthrough (8 lines):** Creates `Celery("learner", broker/backend=REDIS_URL)` and sets `task_routes = {"trigger_learning": {"queue":"learning"}}`.
**⚠️ Critical gap:** it **does not define** `trigger_learning` (no `@app.task`). A worker started with `-A worker -Q learning` therefore has no handler for incoming `trigger_learning` messages and will raise `NotRegistered`. The task body exists only in `services/webhook/worker.py`, which is not in this image. **Fix:** define `trigger_learning` here (calling `learner:8004/learn` or the learn logic directly), or point the learner-worker at a module that registers it. See §17.

#### Per-service `requirements.txt` (5 files)

| File | Notable deps (beyond fastapi/uvicorn/httpx/pydantic-settings/prometheus-fastapi-instrumentator) | Why |
|---|---|---|
| `gateway` | *(none extra)* | Only needs HTTP + HMAC (stdlib). |
| `webhook` | `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `celery`, `redis` | DB + queue + migrations (image also runs the migrate Job). |
| `orchestrator` | `sqlalchemy[asyncio]`, `asyncpg`, `langgraph`, `langchain-openai`, `langfuse`, `openai`, `PyJWT`, `cryptography` | Agents + tracing + GitHub App JWT. |
| `reviewer` | `sqlalchemy[asyncio]`, `asyncpg`, `PyJWT`, `cryptography` | DB + GitHub App JWT. |
| `learner` | `sqlalchemy[asyncio]`, `asyncpg`, `celery`, `redis` | DB + queue. |

**Risk:** all are **unpinned** (no `==` versions) — non-reproducible builds; a transitive/major bump (e.g. LangGraph, OpenAI, Pydantic v1↔v2) could break a build silently. The only pinned deps are in `scripts/Dockerfile`.

#### Per-service `Dockerfile` (5 service images + 1 evaluate image)

All five service Dockerfiles share this shape (build context = repo root):

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY services/<svc>/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY services/<svc>/ .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "<port>"]
```

| Image | Port | Delta |
|---|---|---|
| gateway | 8000 | baseline. |
| webhook | 8001 | **also** `COPY db/ /db/` (so the migrate Job can run Alembic from this image). |
| orchestrator | 8002 | baseline (heaviest deps). |
| reviewer | 8003 | baseline. |
| learner | 8004 | baseline (workers override `CMD` with a Celery command in their k8s manifest). |

**Notes/risks:** no non-root `USER`, no multi-stage build, no healthcheck in the image (k8s probes cover it), copies the whole service dir (incl. `deploy.txt`). `:slim` keeps size down; `--no-cache-dir` avoids pip cache bloat.

#### `services/*/deploy.txt` (5 files)

**Role:** A throwaway file whose *only* purpose is to be edited to trigger that service's path-filtered pipeline (`paths: ["services/<svc>/**"]`) without touching real code. All five currently contain `8` (with a UTF-8 BOM). `INSTRUCTIONS.md` §13 explains the `for %s in (...) do echo N > services\%s\deploy.txt` trick to fan out a redeploy.
**Why it matters:** Clever, low-tech way to force a deploy of one or all services. **Risk:** none functional; purely an operational convenience. The BOM is a Windows `echo`/redirect artifact.

### 10.2 Database (`db/`)

#### `db/alembic.ini`

**Role:** Alembic configuration. `script_location = migrations`, `prepend_sys_path = .`, a default sync `sqlalchemy.url` (`postgresql+asyncpg://user:password@postgres:5432/codereviewer`), and logging config.
**Why it matters:** Drives the `db-migrate` Job. **Key settings:** the URL here is a placeholder; `env.py` overrides it from `DATABASE_URL`. Loggers are set to `WARN` (root/sqlalchemy) and `INFO` (alembic).
**Note/edge case:** the default URL embeds `asyncpg`, which only works because `env.py` uses the async engine path; a stock sync Alembic setup would choke on `+asyncpg`.

#### `db/migrations/env.py`

**Role:** Async-aware Alembic runner.
**Walkthrough:** Reads `DATABASE_URL` from env (falling back to the INI URL) and sets it on the config. `target_metadata = None` → **autogenerate is not used**; migrations are hand-written. `run_migrations_online` calls `asyncio.run(run_async_migrations())`, which builds an async engine with `NullPool`, opens a connection, and runs `do_run_migrations` via `connection.run_sync`. Offline mode emits SQL with literal binds.
**Interview point:** explains how Alembic runs against an async driver. **Risk:** with `target_metadata=None`, `alembic revision --autogenerate` won't detect model drift — schema changes must be written by hand.

#### `db/migrations/versions/0001_initial.py`

**Role:** The one and only migration; creates the schema.
**Walkthrough:** `revision="0001"`, `down_revision=None`. `upgrade()` runs `CREATE EXTENSION IF NOT EXISTS "pgcrypto"` then creates `pull_requests`, `findings` (with FK to `pull_requests.id`), and `patterns` (with `UNIQUE(repo_full_name, pattern_text)`). UUID PKs default to `gen_random_uuid()` (server-side). `downgrade()` drops the three tables in FK-safe order.
**Interview point:** server-side UUID defaults via pgcrypto; the unique constraint exists specifically to power the learner's upsert. **Gap:** no index on `findings.pr_id` (queried by the learner and evaluate) or on `pull_requests (repo_full_name, pr_number, head_sha)` (queried by webhook dedupe) — fine at demo scale, a scan risk later.

### 10.3 Scripts (`scripts/`)

#### `scripts/evaluate.py`

**Role:** Offline quality gate. Pulls recent findings and scores them with RAGAS.

**Detailed walkthrough:**

| Section | What it does | Inputs | Outputs | Side effects | Notes/edge cases |
|---|---|---|---|---|---|
| URL rewrite + connect (L11–13) | `DATABASE_URL` → drop `+asyncpg` → psycopg2 connect. | env | conn | DB connection | RAGAS/psycopg2 are sync, hence the rewrite. |
| query (L15–26) | Last 50 findings joined to PRs, newest first. | — | rows | DB read | Empty → print + `exit(0)` (treated as pass). |
| build dataset (L32–43) | question="What issues exist in this code?", answer=message, contexts=`[file]`. | rows | HF `Dataset` | none | ⚠️ context is only the **filename**, not the diff — weak grounding for faithfulness. |
| evaluate (L44–51) | RAGAS `faithfulness` + `answer_relevancy`; print; compute mean faithfulness. | dataset | scores | OpenAI calls (RAGAS uses an LLM judge) | Defaults faithfulness to `1.0` if the column is missing. |
| gate (L53–55) | `exit(1)` if mean faithfulness < 0.7. | score | exit code | — | This is the CI gate. |

**Interview points:** RAGAS metrics meaning; why the URL rewrite; the deliberate "no data = pass" choice. **Risks:** needs `OPENAI_API_KEY` (RAGAS judges via an LLM); filename-only context limits validity; 50-row cap is arbitrary; answer_relevancy is computed but not gated.

#### `scripts/Dockerfile`

**Role:** Builds the `evaluate` image. **Walkthrough:** `python:3.11-slim`; installs **pinned** `ragas==0.1.21`, `langchain-openai==0.1.25`, plus `psycopg2-binary`, `datasets`; copies `evaluate.py`; `CMD ["python","evaluate.py"]`.
**Note:** the only place in the repo with pinned versions — sensible, because RAGAS/LangChain APIs move fast.

### 10.4 Monitoring (`monitoring/`)

#### `monitoring/prometheus.yml`

**Role:** Standalone Prometheus scrape config. `scrape_interval: 15s`; five static `job_name`s (gateway:8000 … learner:8004), each scraping `/metrics`.
**Why it matters:** Tells Prometheus where the services' metrics live. **Note:** these are **static** targets using k8s service DNS (short names) — works when Prometheus runs in the same namespace. (The runbook actually installs Prometheus via Helm and applies this as a ConfigMap; the chart's own k8s service discovery may differ from these static entries.)

#### `monitoring/grafana-dashboard.json`

**Role:** An importable Grafana dashboard (schemaVersion 38, uid `ai-code-reviewer`).
**Why it matters:** The operator's at-a-glance health view. **Structure:** a templated Prometheus datasource var; **5 service rows** (Gateway/Webhook/Orchestrator/Reviewer/Learner) each with 3 timeseries panels — **Request Rate** (`sum(rate(http_requests_total{app="X"}[5m])) by (method,handler,status)`), **p99+p50 Latency** (`histogram_quantile(…http_request_duration_seconds_bucket…)`), **Error Rate** (`…status=~"5.."`); plus an **Overall Health** row with 4 stat panels (total req/s, total 5xx/s, overall p99, running pods via `count(up{job="kubernetes-pods",namespace="default"}==1)`). Refresh 30s, window `now-1h`.
**Treated as:** a first-party generated/config artifact — documented at panel level, not line-by-line JSON. **Gotcha:** panels query by an `app="…"` label; the metrics from `prometheus-fastapi-instrumentator` are labeled `http_requests_total{...}` but the `app` label must be attached by Prometheus relabeling (from pod labels). If scraping uses the static `prometheus.yml` (no pod-label relabel), the `app` label may be absent and panels could read empty — *worth verifying in a live cluster (Unclear from repo evidence which scrape path was actually used)*.

### 10.5 Infrastructure — Terraform (`infra/terraform/`)

#### `infra/terraform/main.tf`  ⭐ (the cloud)

**Role:** Declares the entire AWS footprint.

**Resource-by-resource:**

| Resource / module | What it creates | Key settings | Notes/risks |
|---|---|---|---|
| `provider "aws"` + `data aws_caller_identity` | Region binding + account id | `region = var.region` | account id used for S3 bucket name + EKS access entry ARN. |
| `module.vpc` (terraform-aws-modules/vpc ~>5) | VPC `10.0.0.0/16`, **public subnets only** in 2 AZs | `enable_nat_gateway=false`, `map_public_ip_on_launch=true`, default SG locked (no ingress/egress) | ⚠️ Cost choice: workloads in **public** subnets (no private subnets + NAT). ELB tags on subnets for the LB controller. |
| `module.eks` (terraform-aws-modules/eks ~>20) | EKS `1.32`, public endpoint, creator admin, an **access entry** for `github-actions-ai-reviewer` (cluster admin), node SG rules for :80 and :3000 (Grafana NLB), one managed node group (`t3.medium`, min1/desired2/max3) | `enable_cluster_creator_admin_permissions=true` | Public API endpoint; cluster-admin for the CI role (broad). |
| `aws_security_group.rds` / `.elasticache` | SGs allowing 5432 / 6379 from `10.0.0.0/16` | `create_before_destroy` | VPC-internal access only. |
| `aws_db_subnet_group` + `aws_db_instance.postgres` | RDS Postgres 15, `db.t3.micro`, 20 GB, `codereviewer` DB, user `dbadmin` | `multi_az=false`, `publicly_accessible=false`, `skip_final_snapshot=true` | ⚠️ single-AZ, no final snapshot (data lost on destroy) — fine for a demo. Password from `var.db_password`. |
| `aws_elasticache_subnet_group` + `aws_elasticache_cluster.redis` | Redis 7.0, `cache.t3.micro`, 1 node | `default.redis7` | Single node, no replication/auth token. |
| `aws_iam_policy.lbc` + attachment | IAM policy for the **AWS Load Balancer Controller**, attached to the node group role | `elasticloadbalancing:*`, EC2 SG/tags, ACM/WAF/Shield reads | Attaching to the **node role** (vs IRSA) is simpler but coarser. |
| `aws_s3_bucket.reports` | `ai-code-reviewer-reports-<acct>` | tags | Declared; no tracked runtime writer. |
| `aws_ecr_repository.*` ×6 | gateway, webhook, orchestrator, reviewer, learner, **evaluate** | `force_delete=true`, `MUTABLE` | `force_delete` lets `terraform destroy` remove repos even with images. |

**Interview points:** module-based VPC/EKS; OIDC access entry instead of stored keys; deliberate cost tradeoffs (no NAT, micro DB/cache); `force_delete` ECR for clean teardown. **Risks:** public subnets, public EKS endpoint, cluster-admin CI role, single-AZ DB, no state backend declared (local `tfstate`, which `.gitignore` excludes).

#### `infra/terraform/variables.tf`

**Role:** Inputs — `region` (default `ap-southeast-2`), `cluster_name` (required), `db_password` (required, `sensitive=true`), `environment` (default `staging`).
**Note:** the README/runbook pass `environment=production`; default is `staging`. `db_password` marked sensitive so it won't print in plan output.

#### `infra/terraform/outputs.tf`

**Role:** Exposes `eks_cluster_endpoint`, `rds_endpoint`, `redis_endpoint` (host:port), and ECR URLs for gateway/webhook/orchestrator/reviewer/learner.
**Note:** **no `evaluate` ECR output** even though the repo is created — operators read it from the AWS console or derive it. These outputs feed the ConfigMap/Secret filling in §15 of the runbook.

### 10.6 Infrastructure — Kubernetes (`infra/k8s/`)

#### `infra/k8s/configmap.yaml`

**Role:** Non-secret runtime env (`app-config`), mounted via `envFrom` into every pod.
**Keys:** `LANGFUSE_HOST` (`https://cloud.langfuse.com`), `REDIS_URL`, and four service URLs (`WEBHOOK/ORCHESTRATOR/REVIEWER/LEARNER_SERVICE_URL`).
**⚠️ Observation:** the tracked `REDIS_URL` is a **real-looking ElastiCache endpoint** (`ai-code-reviewer-redis.hlvcbj.0001.apse2.cache.amazonaws.com:6379`). It is an *internal* AWS hostname (not reachable from the internet) for infra that has been torn down, so it is not a credential — but committing it is minor hygiene leakage (reveals naming/region). The `*_SERVICE_URL` entries are defined but the Python actually hardcodes the same URLs, so they're informational/unused by the code.

#### `infra/k8s/secret.yaml`

**Role:** A **placeholder** `Opaque` Secret (`app-secrets`) with empty `stringData` values for all 7 secret keys.
**Why it matters:** Documents the secret contract while keeping real values out of git (the real file is the gitignored `secret.local.yaml`). Comments point to `kubectl create secret`, Sealed/External Secrets, or SOPS as injection options.
**Security note:** correctly contains **no secret values** (`[REDACTED]`/empty). This is the right pattern.

#### Service Deployments: `gateway.yaml`, `webhook.yaml`, `orchestrator.yaml`, `reviewer.yaml`, `learner.yaml`

All five share one template (documented once, deltas below):

- **Deployment:** `replicas: 2`; pod template labeled `app: <svc>` with Prometheus scrape annotations (`prometheus.io/scrape:"true"`, `…/port:"<port>"`, `…/path:"/metrics"`); one container `image: ${ECR_REGISTRY}/<svc>:latest`; `envFrom` the ConfigMap + Secret; resources `requests 200m/256Mi`, `limits 500m/512Mi`; **liveness** + **readiness** probes hitting `/health`.
- **Service:** `ClusterIP` mapping `port == targetPort == <port>`.

| File | Port | Deltas |
|---|---|---|
| `gateway.yaml` | 8000 | **Adds a second Service `gateway-lb` of type `LoadBalancer`** (port 80→8000) in addition to the ClusterIP — a direct LB path alongside the ALB ingress. |
| `webhook.yaml` | 8001 | baseline. |
| `orchestrator.yaml` | 8002 | baseline (its scaling is handled by `hpa.yaml`). |
| `reviewer.yaml` | 8003 | baseline. |
| `learner.yaml` | 8004 | baseline. |

**Notes/risks:** `image: …:latest` means rollouts rely on `kubectl set image …:$GITHUB_SHA` (CI) to pin a real digest; a bare `kubectl rollout restart` would re-pull `:latest` (mutable tag → reproducibility risk). The `${ECR_REGISTRY}` literal must be substituted at apply time (envsubst/sed) — applying the file verbatim would fail. No `securityContext`/non-root, no pod anti-affinity, no `PodDisruptionBudget`.

#### Worker Deployments: `webhook-worker.yaml`, `learner-worker.yaml`

- Reuse the **service image** (`webhook` / `learner`) but override `command: ["celery","-A","worker","worker","--loglevel=info","-Q","<queue>"]`.
- `webhook-worker`: `replicas: 2`, queue `webhook`. `learner-worker`: `replicas: 1`, queue `learning`.
- **Liveness** uses `celery -A worker inspect ping` (no readiness/HTTP, no Prometheus annotations — workers don't serve HTTP).
- **⚠️** As noted, the `learner-worker`'s image (`services/learner/`) does not register `trigger_learning`, so this worker likely cannot execute the queued task (see §6/§17).

#### `infra/k8s/hpa.yaml`

**Role:** `HorizontalPodAutoscaler` for the orchestrator. `minReplicas: 2`, `maxReplicas: 6`, target **70% CPU utilization** (`autoscaling/v2`).
**Why it matters:** The orchestrator is the CPU/latency-heavy service (LLM calls), so it's the one that scales. **Note:** HPA needs `metrics-server` installed (not in the repo; assumed in-cluster) and the resource `requests` (200m) define the utilization base.

#### `infra/k8s/ingress.yaml`

**Role:** ALB ingress (`ingressClassName: alb`) routing `/` → `gateway:8000`.
**Annotations:** `scheme: internet-facing`, `target-type: ip`, `listen-ports: [{"HTTP":80}]`, `healthcheck-path: /health`.
**⚠️ Risk:** **HTTP only** (no HTTPS/ACM cert) — GitHub webhooks travel unencrypted (acknowledged as a demo tradeoff). Requires the AWS Load Balancer Controller to materialize the ALB.

#### `infra/k8s/migration-job.yaml`

**Role:** One-shot `Job` `db-migrate` using the **webhook** image (which bundles `db/`), `workingDir: /db`, `command: alembic -c /db/alembic.ini upgrade head`, with `DATABASE_URL` from the secret. `backoffLimit: 3`, `restartPolicy: Never`.
**Why it matters:** Creates the schema before services need it. **Note:** uses `:latest` implicitly via the image ref (the manifest as written would also need `${ECR_REGISTRY}` substitution).

#### `infra/k8s/evaluate-job.yaml`

**Role:** Template `Job` `evaluate` with `image: IMAGE_PLACEHOLDER` (substituted by `sed` in the Actions workflow), env `DATABASE_URL` + `OPENAI_API_KEY` from the secret, `backoffLimit: 0`, `restartPolicy: Never`.
**Why it matters:** The RAGAS job the weekly pipeline applies. `backoffLimit: 0` = no retries (a failed eval fails the run cleanly).

#### `infra/k8s/kubectl`

**Role / status:** A **0-byte, empty, tracked file** named `kubectl` in `infra/k8s/`. It is **not** the kubectl binary (0 bytes) and is referenced by nothing.
**Assessment:** Almost certainly an **accidental commit** (e.g. a stray `> kubectl` redirect or an interrupted binary download). Line-by-line analysis is N/A (no content). **Recommendation:** delete it (`git rm infra/k8s/kubectl`). Documented here for completeness and included in the coverage checklist.

### 10.7 CI/CD (`.github/workflows/`)

#### `.github/workflows/service-ci.yml`  ⭐ (reusable pipeline)

**Role:** The shared 3-job pipeline, invoked per service via `workflow_call` with a `service` input.
**Permissions:** `id-token: write` (OIDC), `contents: read`.

| Job | Runs when | Steps |
|---|---|---|
| `test` | always | checkout; setup Python 3.11; `pip install -r requirements.txt` in `services/<svc>`; `if [ -d tests ]; then pytest; fi`. |
| `build-and-push` | `push` to `main` only | checkout; `configure-aws-credentials` (assume `AWS_ROLE_ARN`, region `ap-southeast-2`); ECR login; `docker build -f services/<svc>/Dockerfile .` tagged `:$GITHUB_SHA` + `:latest`; push both. |
| `deploy` | `push` to `main` only | checkout; assume role; `aws eks update-kubeconfig`; `kubectl set image deployment/<svc> <svc>=…:$GITHUB_SHA`. |

**Interview points:** reusable workflow DRY; OIDC keyless auth; immutable `$GITHUB_SHA` tag for the actual rollout (with a convenience `:latest`); path-filtered fan-out. **Risks:** the `test` job's conditional makes "tests" a no-op today; `deploy` only updates the same-named **Deployment** (not workers or the `db-migrate`/`evaluate` Jobs); region hardcoded in three places.

#### `.github/workflows/{gateway,webhook,orchestrator,reviewer,learner}.yml` (5 triggers)

Identical except the service name. Each: `on push`/`pull_request` to `main` filtered to `paths: ["services/<svc>/**"]`; `permissions: id-token/contents`; one job `ci` that `uses: ./.github/workflows/service-ci.yml with: service=<svc>` and `secrets: inherit`.
**Why it matters:** This is what makes a change to one service redeploy **only** that service. (A `pull_request` runs only the `test` job, since build/deploy gate on `push` to `main`.)

#### `.github/workflows/evaluate.yml`

**Role:** The weekly evaluation pipeline. `on: schedule (cron "0 9 * * 1")` + `workflow_dispatch`. Steps: checkout → assume AWS role (OIDC) → ECR login → build & push the `evaluate` image → `aws eks update-kubeconfig` → delete any old `evaluate` Job → `sed` `IMAGE_PLACEHOLDER` and `kubectl apply` the Job → `kubectl wait` for complete/failed (600s) → print logs (`if: always()`) → cleanup the Job (`if: always()`).
**Interview point:** running a batch ML eval as a k8s Job from CI; idempotent delete-before-apply; `always()` for logs/cleanup. **Note:** the `wait` uses both `--for=condition=complete` and `--for=condition=failed` so it returns on either outcome.

### 10.8 Root documentation & config

#### `README.md`

**Role:** The canonical narrative (178 lines): badges, a mermaid architecture diagram, "what it does," a capability-to-role mapping table, "how it works," key decisions/tradeoffs, results, tech stack, repo layout, a 6-step "how to run," and an honest limitations list.
**Why it matters:** The single best high-level source; this `ALLINFO` expands and verifies it against code. **⚠️ Dangling assets:** it embeds `docs/pr-review.png` and `docs/langfuse-dashboard.png`, but there is **no tracked `docs/` directory** — these images will not render on GitHub. The linked `ai-code-review-pr-screenshot.pdf` *is* tracked. **Accuracy:** README's self-reported bugs (double-emit, inline fallback, no tests) all match the code; it does **not** mention the learner-worker task-registration gap (a finding surfaced in this document).

#### `INSTRUCTIONS.md`

**Role:** A 1,881-line, Windows-CMD-oriented production runbook: 16 phases from account creation → tool install → AWS/OIDC setup → GitHub App → LangFuse/OpenAI keys → `terraform apply` → GitHub secrets → first deploy → kubectl/EKS → applying manifests **in a specific order** → Prometheus/Grafana (incl. EBS CSI driver) → LangFuse tracing → webhook URL → end-to-end test → dashboards → weekly eval → health checks → troubleshooting → costs (~$179/mo fixed) → teardown ordering (avoid `DependencyViolation`).
**Why it matters:** The operational bible; the source for §3 and §13 here. **Notes:** it carries **leftover personal paths** (`D:\MAJOR PROJECT KRISH SIR\…`), a couple of **personal/old URLs** (e.g. a `data-guru0/MAJOR-PROJECT-KRISH-SIR` Actions link, a concrete ALB hostname, region drift between `us-east-1` examples and the Terraform default `ap-southeast-2`). These are documentation inconsistencies, not code bugs, but worth cleaning before sharing. No secret *values* are committed (placeholders only).

#### `LICENSE`

**Role:** MIT License, © 2026 Srikara S. Standard permissive terms. Enables reuse with attribution.

#### `.gitignore`

**Role:** Ignore rules. Protects `.env`/`.env.*` (allow-listing `.env.example`), Python artifacts, `node_modules`, Terraform state/plans/lockfile, `*.kubeconfig`, IDE/OS files, logs, and crucially **secrets**: `*.pem/*.key/*.crt`, `secrets/`, `infra/k8s/secret.local.yaml`, `infra/k8s/secrets/`.
**Why it matters:** The first line of secret-leak defense; the comments explicitly note that the tracked `secret.yaml` is a placeholder and real values live in the ignored `secret.local.yaml`. **Note:** it ignores Terraform's `.terraform.lock.hcl`, which is normally *committed* for reproducible provider versions — a minor IaC hygiene gap.

#### `ai-code-review-pr-screenshot.pdf`

**Role / status:** A **tracked binary** (≈955 KB) — a printout of the bot posting a consolidated review on a sample PR (the "proof" artifact referenced by README).
**Why detailed analysis is N/A:** It's a binary PDF, not source; its value is as evidence of a real run, not as code. Documented by path/role/tracked-status per the binary-file rule. (It is also the largest file and the only non-text tracked blob.)

---

## 11. Cross-Cutting Concerns

**Security & secrets handling.** Strong inbound auth (HMAC-SHA256, constant-time) and outbound auth (GitHub App JWT→installation token, short-lived). No long-lived AWS keys (OIDC). Secrets kept out of git via `.gitignore` + placeholder `secret.yaml` (real values in `secret.local.yaml`). *Weak points visible in repo:* HTTP-only ingress (`ingress.yaml`), public EKS endpoint + public subnets (`main.tf`), cluster-admin CI role, a committed internal Redis hostname (`configmap.yaml`), and `gateway` exposing an extra `LoadBalancer` Service in addition to the ALB. No rate limiting, no request size limits on the diff.

**Error handling.** Inconsistent. Outbound calls in the gateway/orchestrator(token,diff)/reviewer use `raise_for_status()`, but the orchestrator's call to the reviewer and both Celery tasks do **not**, so some downstream failures are swallowed. The JSON parser returns `[]` on error (silent finding loss). No retry/back-off/circuit-breaker; the Celery task `timeout` is the only guard. The merge-handler `scalar_one_or_none()` can raise on legitimate multi-commit PRs.

**Logging & observability.** Metrics: `prometheus-fastapi-instrumentator` on all five services + a Grafana dashboard. LLM tracing: LangFuse on every model call (prompt/response/tokens/cost/latency). *Gaps:* no application logging (only framework stdout via `kubectl logs`), no cross-service correlation IDs, no alerting rules (dashboards only), workers expose no metrics.

**Testing strategy.** Effectively none today — see §12.

**Performance.** Agents run concurrently (`Send` fan-out), so a review ≈ slowest agent (~5s). *Costs:* the synchronous `build_graph().invoke()` inside the async orchestrator handler blocks the event loop; the graph is compiled per request; the whole diff is sent to each agent with no token budgeting; the double-emit bug doubles DB writes and posted content; new `httpx` clients per call (no pooling).

**Scalability.** HPA on the orchestrator (2–6) is the headline; queue-based decoupling smooths webhook bursts. *Limits:* single-AZ `db.t3.micro` and single-node `cache.t3.micro`; no DB indexes for the hot lookups; learner-worker is a single replica; no PodDisruptionBudgets.

**Accessibility.** N/A — no frontend.

**Data privacy.** PR diffs (potentially proprietary code) are sent to OpenAI and traced in LangFuse; findings (which may quote code) are stored in Postgres and surfaced to RAGAS/OpenAI during evaluation. This is an important disclosure point for any real adoption. No PII handling beyond repo/PR identifiers.

**Dependency management.** Per-service `requirements.txt`, **unpinned** (reproducibility risk) except the evaluate image. No lockfiles. Terraform pins module/provider major versions; `.terraform.lock.hcl` is gitignored (should be committed).

**Code organization.** Clean per-service folders; small, readable files. *Tradeoff:* ORM models + `Settings` are **duplicated** across services (no shared package) — simple to deploy independently, but drift-prone (already a cosmetic `status` difference).

**Maintainability.** High for the size; the agent design makes adding a dimension a localized change. The duplication and lack of tests are the main debts.

**Deployment readiness.** Strong scaffolding (IaC, per-service CI/CD, probes, HPA, migrations-as-Job, monitoring). Not production-hardened (HTTPS, private networking, multi-AZ, secrets manager integration, the known bugs).

**Failure modes (most likely first).** (1) learner-worker `NotRegistered` → learning loop never runs; (2) multi-commit PR merge → `MultipleResultsFound`; (3) doubled findings; (4) inline `422` → summary-only; (5) malformed/oversized diff → empty/poor findings; (6) OpenAI quota/key → no review; (7) `:latest` re-pull drift.

**Technical debt.** No tests; unpinned deps; duplicated models; the three known code bugs; the stray `infra/k8s/kubectl`; dangling README images; runbook personal paths + region drift; gitignored TF lockfile.

---

## 12. Testing And Validation

- **Frameworks present:** `pytest` is invoked by CI, but **no test files exist** anywhere in the repo (no `tests/` dirs, no `test_*.py`). The CI guard `if [ -d tests ]; then pytest; fi` makes the test step a deliberate no-op.
- **What's covered:** Nothing automatically. Validation was **manual/end-to-end**: per README, a real PR with a deliberately flawed file (`password="admin123"`, `eval()` over concatenated input) was reviewed in ~1 minute; LangFuse recorded ~$0.0007 for the four-agent run; `terraform destroy` removed 66 resources to $0. The `ai-code-review-pr-screenshot.pdf` is the artifact.
- **What's untested:** HMAC verification; dedupe logic; the LangGraph wiring (incl. the double-emit); JSON parsing fallback; token minting; the `422` fallback; the upsert; the Celery routing/registration; the merge-handler `scalar_one_or_none()` path; the RAGAS gate.
- **How to run (today):** `cd services/<svc> && pip install -r requirements.txt && pytest` — exits cleanly with "no tests ran."
- **Utilities/mocks/fixtures:** none.
- **High-value tests to add (project-specific):**
  1. `test_graph_dedup` — assert `build_graph().invoke()` returns each finding **once** (this would have caught the `operator.add` double-emit).
  2. `test_hmac` — gateway accepts a correctly-signed body and `401`s a tampered one.
  3. `test_dedupe` — second identical `(repo,pr,sha)` event → `already_processing`; a `synchronize` with a new sha → new row.
  4. `test_merge_handler_multi_commit` — two rows for one PR then merge → must not raise (drives a `scalar_one_or_none` fix).
  5. `test_learner_task_registered` — assert the learner Celery app has `trigger_learning` registered (guards the §17 fix).
  6. `test_parse_json_response` — fenced/bare/garbage inputs.
  7. `test_reviewer_422_fallback` — mock GitHub `422` → asserts summary-only retry.
  8. `test_upsert_frequency` — two merges → `frequency==2`.

---

## 13. Build, Deployment, And Operations

**Build.** Per-service Docker images (`python:3.11-slim`, pip install, copy, `uvicorn`/`celery`). Built **in CI** (GitHub Actions), not locally. Tagged `:$GITHUB_SHA` + `:latest`, pushed to ECR. The evaluate image is built separately by `evaluate.yml`.

**Runtime.** EKS runs the 7 workloads (5 services ×2 + webhook-worker ×2 + learner-worker ×1) plus Jobs. Services serve via Uvicorn; workers run Celery; RDS/ElastiCache are managed; the ALB (via the LB controller) fronts the gateway.

**Deployment clues.** `infra/terraform` (stack), `infra/k8s` (workloads, applied in the runbook's exact order), `.github/workflows` (pipelines), `INSTRUCTIONS.md` (the 16-phase runbook), `services/*/deploy.txt` (redeploy triggers), and the git history (`deploy: trigger initial deployment`, `deploy: re-trigger all 5 pipelines`, `fix: align CI/CD and Terraform region to ap-southeast-2`).

**Docker/k8s/cloud.** Covered in §10.5–10.6. Highlights: HPA on orchestrator; ALB ingress (HTTP); migration-as-Job; Helm-installed AWS LB Controller + Prometheus + Grafana (operational, not tracked as charts).

**CI/CD.** Five path-filtered per-service pipelines + a reusable 3-stage workflow + a weekly eval pipeline, all OIDC-authenticated. See §10.7.

**Monitoring/logging.** Prometheus scrape (`monitoring/prometheus.yml`) + Grafana dashboard (`monitoring/grafana-dashboard.json`); LangFuse for LLM traces; `kubectl logs` for stdout. No alerting rules in-repo.

**Operational risks.** `:latest` mutability; HTTP ingress; single-AZ DB with `skip_final_snapshot` (destroy = data loss); cluster-admin CI role; no alerts; the learner-worker registration gap silently breaks learning; teardown requires the documented ordering to avoid `DependencyViolation`.

**Debugging a production incident from this codebase.**
- *No reviews posting:* `kubectl logs -l app=orchestrator -f` and `-l app=reviewer -f`; check OpenAI key/credits, GitHub App permissions (PR read/write), PEM line breaks; verify the webhook delivered (GitHub App → Advanced → Recent Deliveries; `404`=path, `401`=secret).
- *Duplicated comments:* expected (double-emit bug) — fix in `graph.py`.
- *Learning not improving reviews:* inspect `learner-worker` logs for `NotRegistered`; check the `patterns` table is being written; confirm the §17 fix.
- *Latency/5xx spike:* Grafana per-service panels; `kubectl top pods`; check HPA (`kubectl get hpa`).
- *Eval failures:* Actions "Evaluate" run logs; LangFuse traces for off-topic/broken-JSON responses → tune prompts in `graph.py`.

---

## 14. How To Modify Or Extend This Project

**Add a new review dimension (agent).** In `services/orchestrator/graph.py`: add a prompt to `PROMPTS` (or a `_xxx_prompt(state)` builder for a dynamic one); `builder.add_node("xxx", make_node("xxx", PROMPTS["xxx"]))`; add `Send("xxx", state)` to `fan_out`; add `builder.add_edge("xxx", "merge")`. No schema change needed (findings are generic). Then bump `services/orchestrator/deploy.txt` (or edit any orchestrator file) and push → only the orchestrator redeploys.

**Add a new HTTP endpoint to a service.** Add an `async def` handler decorated on that service's `app` in `main.py`, reusing the existing `AsyncSessionLocal`/`Settings` pattern. Keep `/health` + `/metrics` intact (probes/scraping depend on them).

**Add a new service.** Create `services/<new>/{main.py,models.py,requirements.txt,Dockerfile,deploy.txt}`; add a k8s Deployment+Service (copy an existing manifest, change name/port/image); add an ECR repo in `main.tf`; add `.github/workflows/<new>.yml` (copy a trigger, change `service:` + `paths:`); add a Prometheus job + Grafana row if it serves HTTP.

**Add a new data model / migration.** Add the ORM class in the relevant service(s)' `models.py` (remember the duplication), then write a **hand-authored** Alembic revision under `db/migrations/versions/` (autogenerate is off — `target_metadata=None`). Apply via the `db-migrate` Job (or `alembic upgrade head`). Consider extracting models into a shared package to end the duplication.

**Add a new background task.** Define `@app.task(name="...")` in the appropriate `worker.py`, add a `task_routes` entry, enqueue with `.apply_async(args=[...], queue="...")`, and ensure the **consuming** worker's image actually registers that task (the existing learner gap is the cautionary tale).

**Add tests.** Create `services/<svc>/tests/` with `test_*.py`; CI will then run `pytest` for that service automatically. Add `pytest` (and `pytest-asyncio`, `httpx`/`respx` for mocking) to that service's `requirements.txt`. Start with the list in §12.

**Debug common issues.** Use the §13 playbook + `INSTRUCTIONS.md` §23. For LLM-quality issues, read LangFuse traces and tune `graph.py` prompts; for JSON breakage, harden `parse_json_response`.

**Avoid breaking existing patterns.** Keep `/health` + `/metrics` on every service; keep service-to-service URLs as k8s DNS names; preserve the `202`-and-enqueue contract; don't add secret values to tracked files (use `secret.local.yaml`); keep one ECR repo + one pipeline per service; tag images with `$GITHUB_SHA` for real rollouts.

---

## 15. Interview Preparation Pack

### 15.1 Elevator Pitches

**30-second.** "I built an AI pull-request reviewer that runs as seven microservices on Kubernetes. When a PR opens, four specialized LLM agents — static analysis, security, style, architecture — review the diff in parallel and post one consolidated review. It's got LangFuse tracing, a RAGAS evaluation gate, per-service CI/CD over OIDC, and full Terraform infra on AWS. I deployed it, validated it on a real PR for under a cent, then tore it down to $0."

**60-second.** Add: "The interesting engineering isn't the LLM call — it's everything around it. A gateway does HMAC verification only; a webhook service persists the PR and drops a Celery job on Redis so it can return 202 immediately; a worker calls an orchestrator that mints a GitHub App installation token, fetches the diff, and runs a LangGraph graph that fans out to four `gpt-4o-mini` agents and merges their findings. A reviewer posts the result, with a fallback to a summary comment when the model's line numbers don't anchor to the diff. On merge, a learner distills recurring issues into a per-repo pattern table that conditions future reviews. Every model call is traced, a weekly RAGAS job gates on faithfulness, and an HPA scales the orchestrator on CPU."

**2-minute technical.** Add the data model (`pull_requests`/`findings`/`patterns` with an upsert frequency counter), the LangGraph `Send` fan-out + `operator.add` reducer + merge node, the GitHub App JWT→installation-token exchange, OIDC keyless deploys, the 6 ECR repos and Terraform-managed VPC/EKS/RDS/ElastiCache/S3, and end with honesty: "It's a validated demo, not a service with users. I can point to three real bugs in it — findings get double-emitted because of a LangGraph reducer interaction, the merge handler can throw on multi-commit PRs, and the learner-worker doesn't register the Celery task it's supposed to consume — and I can tell you the one-line fix for each. I'd rather show I understand my own system than pretend it's perfect."

**Recruiter-friendly.** "It's an AI bot that automatically reviews code on GitHub the moment a pull request is opened — like an always-on senior reviewer that catches security holes and bugs first. I built the whole thing end to end: the AI part, the backend, and all the cloud infrastructure to run it on AWS, with monitoring and automated quality checks. It's a portfolio project that shows I can ship an AI feature like a real product, not just a demo notebook."

**Senior-engineer technical.** "Event-driven microservices: an HMAC edge, a 202-and-enqueue webhook with application-level idempotency on `(repo, pr, head_sha)`, Celery/Redis decoupling, and a LangGraph orchestrator doing parallel role-specialized inference with a merge/dedup reducer. GitHub App auth via short-lived installation tokens, OIDC for keyless CI/CD, per-service path-filtered pipelines, HPA, migrations-as-Job, Prometheus/Grafana, and LangFuse + a RAGAS gate for LLMOps. Cost-tuned infra — public subnets, single-AZ micro DB/cache — which I'd flag and change for production. I know its failure modes cold."

### 15.2 Architecture Questions And Answers

**Q: Why microservices for what's essentially "call an LLM on a diff"?**
A: Partly to demonstrate production/MLOps depth, but the boundaries are real: the gateway isolates *"is this from GitHub?"* from business logic; the queue decouples webhook receipt (must be fast, return 202) from slow analysis; per-service pipelines mean an orchestrator fix redeploys only the orchestrator. The honest tradeoff (stated in the README) is that this traffic doesn't *need* a cluster — the split is justified by the boundaries and the learning goal, not load.

**Q: Why four agents instead of one big prompt?**
A: Focus and parallelism. Each agent gets a tight, role-specific instruction set, the four calls run concurrently (so latency ≈ slowest agent, not the sum), and adding a dimension is one more node rather than a prompt rewrite. The cost is four model calls + a merge step. It also mirrors where multi-agent systems are heading.

**Q: How does data flow end to end?**
A: GitHub→ALB→gateway (HMAC)→webhook (persist + enqueue)→Redis→webhook-worker→orchestrator (token+diff+patterns→LangGraph 4 agents→merge→persist findings)→reviewer (post one review)→PR marked reviewed. On merge: webhook→Redis(learning)→learner-worker→learner (upsert patterns). (See the §5 diagram.)

**Q: Where are the bottlenecks?**
A: The orchestrator: it makes four LLM calls and — as written — runs the LangGraph graph **synchronously inside an async handler**, blocking the event loop for the whole run. That's exactly why it's the service with an HPA. The DB lookups (webhook dedupe, learner read) lack indexes, which would bite at scale.

**Q: How would it scale to thousands of PRs/min?**
A: Add DB indexes on the hot keys; make the orchestrator's graph call non-blocking (run in a threadpool or use async LLM clients); scale webhook-workers and orchestrators (HPA already on orchestrator); move RDS to multi-AZ + read replicas; ElastiCache to a replication group; raise EKS node `max_size`; add rate limiting at the gateway; and cap diff size / chunk huge diffs to control token cost and latency.

**Q: What would fail first under load or in prod?**
A: Functionally, the learning loop (learner-worker `NotRegistered`) and the merge handler on multi-commit PRs. Operationally, the single-AZ `db.t3.micro` and single Redis node, and the event-loop blocking in the orchestrator capping per-pod throughput.

**Q: How would you deploy and roll back?**
A: Push to `main` → path-filtered pipeline → image tagged `$GITHUB_SHA` → `kubectl set image …:$GITHUB_SHA`. Rollback = `kubectl rollout undo deployment/<svc>` or set the image back to a previous SHA. I'd stop using the mutable `:latest` for rollouts and rely solely on the SHA tag.

**Q: How is it monitored?**
A: Prometheus scrapes `/metrics` on all five services; a Grafana dashboard shows per-service request rate, p50/p99 latency, 5xx rate, and an overall-health row. LLM calls are traced in LangFuse (tokens/cost/latency). Gaps I'd close: alert rules, worker metrics, and cross-service correlation IDs.

**Q: Why LangGraph specifically?**
A: It models the fan-out/merge as a graph with a typed state and a `Send`-based conditional entry point, so parallel agents + a reconciliation node are first-class. It also made the bug I hit instructive — the `operator.add` state reducer is a LangGraph concept, and understanding it is what lets me fix the double-emit cleanly.

**Q: Why GitHub App auth instead of a PAT?**
A: Apps get scoped, short-lived **installation tokens** minted from a signed JWT, with per-repo permissions and no human-tied credential. It's the correct pattern for a bot acting across installations.

### 15.3 Code-Level Questions And Answers

**Q: Walk me through the double-emit bug.**
A: In `graph.py`, `GraphState.findings` is `Annotated[list[dict], operator.add]`, so every node's returned `findings` is *concatenated* to the accumulator. The four agents fill it with N raw findings. `merge_node` then dedupes to M and returns `{"findings": merged}` — but because of the reducer, that M is **appended** to the existing N, giving N+M. With no exact-duplicate findings, M==N, so everything is emitted exactly twice. **Fix:** return the merged list under a *different* state key (e.g. `merged_findings`) and read that in the orchestrator, or skip the reducer and dedupe just before saving.

**Q: There's a subtle bug in the learning loop. Find it.**
A: `trigger_learning` is defined with `@app.task` only in `services/webhook/worker.py`. But the `learner-worker` Deployment runs `celery -A worker worker -Q learning` against `services/learner/worker.py`, which sets up the Celery app and a route but **never defines the task**. A Celery worker that receives a task name it hasn't registered raises `NotRegistered`, so the queued message can't execute there — the `/learn` call is never made via the queue. **Fix:** define `trigger_learning` in `services/learner/worker.py` (POSTing to `learner:8004/learn`, or calling the learn logic directly).

**Q: What happens on `receive_event` when a PR with multiple commits is merged?**
A: The merge branch looks up the PR with `select(...).where(repo, pr_number)` then `scalar_one_or_none()`. Since each `synchronize` event inserts a new row (different `head_sha`), a PR that got commits after opening has ≥2 rows, and `scalar_one_or_none()` raises `MultipleResultsFound`. **Fix:** `order_by(created_at.desc()).limit(1)` + `scalar_one_or_none()`, or `.first()`.

**Q: Why is the `DATABASE_URL` rewritten in `evaluate.py`?**
A: Services use the async driver (`postgresql+asyncpg://`) with SQLAlchemy; `evaluate.py` uses **psycopg2** (sync) for RAGAS, which doesn't understand the `+asyncpg` suffix, so it strips it to plain `postgresql://`.

**Q: How does the style agent "learn"?**
A: `_style_prompt(state)` injects the repo's top patterns into the system prompt. Those patterns come from the learner's upsert into `patterns` on merge (`ON CONFLICT (repo_full_name, pattern_text) DO UPDATE SET frequency = frequency+1`), and the orchestrator loads the top-10 by frequency before invoking the graph. It's prompt-conditioning, not fine-tuning.

**Q: Why `hmac.compare_digest` instead of `==`?**
A: Constant-time comparison to avoid leaking the signature via timing differences. And the HMAC is computed over the **raw** request bytes — re-serializing the JSON would change the digest and break verification.

**Q: Why does the webhook return `202` before the review exists?**
A: To decouple receipt from work. GitHub expects a fast webhook response; the actual multi-second LLM analysis happens off the request path via Celery. `202 Accepted` is the honest status. (Minor nit: the orchestrator's `/analyze` is also declared `202` but actually does the work inline before responding.)

**Q: What does `parse_json_response` do when the model returns prose or broken JSON?**
A: It strips a ```` ```json ```` fence if present, tries `json.loads`, and on any exception returns `[]` — so that agent contributes no findings. It's robust but silent; a malformed-but-real finding is lost without a trace.

**Q: Why is the OpenAI client imported from `langfuse.openai`?**
A: It's a drop-in wrapper: same API as the OpenAI SDK, but every call is automatically traced in LangFuse (prompt, response, tokens, cost, latency) with zero changes to the call sites.

**Q: What's the purpose of `services/*/deploy.txt`?**
A: A no-op trigger file. Pipelines are path-filtered to `services/<svc>/**`, so bumping the number in `deploy.txt` forces a redeploy of that service without editing real code — handy for the first deploy and for re-running after the k8s Deployments exist.

### 15.4 Debugging Questions And Answers

**Scenario 1 — Reviews show every comment twice.**
- *Symptom:* duplicated findings in the PR and the `findings` table.
- *Cause:* the `operator.add` reducer + `merge_node` returning under `findings` (§15.3).
- *Files:* `services/orchestrator/graph.py` (`GraphState`, `merge_node`), `services/orchestrator/main.py` (persist).
- *Reproduce:* run any PR; count findings = 2× unique.
- *Fix:* write merged to a new key / dedupe before save.
- *Prevent:* add `test_graph_dedup`.

**Scenario 2 — Merged PRs never improve future reviews.**
- *Symptom:* `patterns` table stays empty; style prompt always says "None."
- *Cause:* learner-worker `NotRegistered` for `trigger_learning` (and/or the merge-handler exception).
- *Files:* `services/learner/worker.py`, `services/webhook/worker.py`, `services/webhook/main.py`.
- *Reproduce:* merge a PR; `kubectl logs -l app=learner-worker` shows `Received unregistered task`.
- *Fix:* register the task in the learner worker.
- *Prevent:* `test_learner_task_registered`.

**Scenario 3 — Merge webhook 500s for some PRs.**
- *Symptom:* `closed/merged` events error; learning never enqueues.
- *Cause:* `scalar_one_or_none()` over multiple rows for one PR.
- *Files:* `services/webhook/main.py` merge branch.
- *Reproduce:* open a PR, push another commit (creates a 2nd row), merge.
- *Fix:* select newest row / `.first()`.
- *Prevent:* `test_merge_handler_multi_commit`.

**Scenario 4 — Comments land as one summary, not inline.**
- *Symptom:* review posts but no line-anchored comments.
- *Cause:* GitHub returns `422` because LLM line numbers don't match the diff → summary-only fallback.
- *Files:* `services/reviewer/main.py`.
- *Reproduce:* any PR where the model guesses line numbers.
- *Fix:* ground line numbers against the parsed diff before posting.
- *Prevent:* `test_reviewer_422_fallback` + a diff-grounding util.

**Scenario 5 — No review at all; pods healthy.**
- *Symptom:* PR opens, nothing posts.
- *Likely causes:* invalid/zero-credit OpenAI key; GitHub App missing PR read/write; PEM line breaks lost in the secret; webhook delivery failing (`404` path / `401` secret).
- *Files/tools:* `kubectl logs -l app=orchestrator/-l app=reviewer`; GitHub App → Advanced → Recent Deliveries; OpenAI usage page.
- *Fix:* correct the key/permissions/secret/URL.
- *Prevent:* a startup self-check + an alert on orchestrator 5xx.

**Scenario 6 — Pod `CrashLoopBackOff` after deploy.**
- *Symptom:* a service won't start.
- *Causes:* `DATABASE_URL` not `postgresql+asyncpg://`; missing secret key; import error.
- *Files/tools:* `kubectl logs POD --previous`, `secret.yaml`.
- *Fix:* correct env/secret; redeploy.
- *Prevent:* validate `Settings` at startup; pin deps.

### 15.5 Design Tradeoff Questions And Answers

| Tradeoff | This project's choice | Alternative | Why |
|---|---|---|---|
| Simplicity vs scalability | 7 services + EKS + HPA | a single FastAPI app | Chose to *demonstrate* prod/MLOps depth; boundaries are real but the traffic doesn't require a cluster (stated honestly). |
| Local vs cloud | Cloud-only (build in CI, run on AWS) | docker-compose for local | Showcases the full AWS/MLOps stack; the cost is no easy local dev (no compose/.env.example). |
| Sync vs async | Async FastAPI + Celery decoupling, **but** a sync graph call in the orchestrator | fully async LLM calls | Decoupling is right; the sync `invoke()` is a known wart that blocks the loop. |
| Type safety | Pydantic on request bodies; `findings` left as loose dicts | typed finding model | The model output is inherently loose JSON; validating it harder would also be reasonable. |
| State management | Postgres for durable state; Redis only as broker | event store / no DB | Simple, queryable, migration-controlled. |
| Error handling | `raise_for_status` in some places, swallowed elsewhere | uniform retries + DLQ | Inconsistency is debt; the queue gives a natural place to add retries. |
| Testing | CI scaffold but zero tests | TDD | Honest gap; the pipeline is ready for tests. |
| Model choice | `gpt-4o-mini` everywhere | a larger model | Deliberate cost choice (~$0.0007/review); no quality benchmark claimed. |
| Framework | LangGraph for orchestration | hand-rolled `asyncio.gather` | LangGraph makes fan-out/merge + state first-class (and taught the reducer lesson). |
| Performance | parallel agents (latency ≈ slowest) | sequential | Concurrency is the headline perf win. |
| Infra hardening | cost-tuned (public subnets, single-AZ, HTTP) | private subnets+NAT, multi-AZ, HTTPS | Keeps the demo bill ~$179/mo; explicitly flagged as "things I'd change for real." |

### 15.6 Behavioral / STAR Stories

> Grounded in repo evidence (git history, README, code). Items I can't fully prove from the repo are marked **[suggested framing]**.

**Building the project.**
- *S:* I wanted a portfolio piece proving end-to-end ownership of an LLM feature, not a notebook.
- *T:* Design, build, deploy, validate, and tear down a real AI code reviewer on AWS.
- *A:* Built 5 FastAPI services + 2 Celery workers, a LangGraph 4-agent orchestrator, Postgres schema with Alembic, Terraform for VPC/EKS/RDS/ElastiCache/ECR/S3, per-service OIDC CI/CD, Prometheus/Grafana, LangFuse, and a RAGAS gate.
- *R:* Validated on a real PR in ~1 minute for ~$0.0007, then `terraform destroy`'d 66 resources to $0. Documented the whole thing in a 16-phase runbook.

**Debugging a hard issue (double-emit).**
- *S:* Findings were being posted twice.
- *T:* Find the root cause in the LangGraph flow.
- *A:* Traced it to the `operator.add` reducer on `GraphState.findings` interacting with `merge_node` returning under the same key, so the deduped list was appended rather than replacing.
- *R:* Identified a one-line-class fix (separate state key / dedupe-at-save) and documented it openly in the README rather than hiding it. **[Repo evidence: README "Limitations"; code in graph.py. The *fix* is noted as pending, not yet applied.]**

**Making an architectural decision (multi-agent split).**
- *S:* One prompt vs many.
- *T:* Decide how to structure the review.
- *A:* Chose four role-specialized agents fanned out via LangGraph `Send`, accepting 4× calls + a merge step for focus, parallelism, and extensibility.
- *R:* Adding a dimension is now a localized change; latency ≈ slowest agent.

**Improving reliability (the 422 fallback).**
- *S:* GitHub rejected reviews when LLM line numbers didn't anchor to the diff.
- *T:* Make reviews post reliably anyway.
- *A:* Detect `422` and retry with the summary body only.
- *R:* Reviews always post; documented the proper fix (grounding line numbers) as next step.

**Learning a new tool/framework (the cloud stack).**
- *S:* The model call was familiar; EKS/IAM/OIDC/ECR/ALB/Prometheus were not.
- *T:* Stand up and tear down a full cloud deployment correctly.
- *A:* Learned EKS node groups, IAM roles + OIDC, ECR, the AWS Load Balancer Controller, EBS CSI for Prometheus storage, and the teardown ordering to avoid `DependencyViolation`.
- *R:* "The bulk of the work" per the README; brought it all back to $0 cleanly.

**Handling ambiguity (cost vs correctness on infra).**
- *S:* Production-grade networking would blow the personal budget.
- *T:* Decide what to harden vs defer.
- *A:* Chose public subnets (no NAT), single-AZ micro DB/cache, HTTP ingress — and **explicitly documented** each as a demo tradeoff with the production alternative.
- *R:* A defensible, transparent posture rather than silent shortcuts.

**Testing/validation.** **[suggested framing]** The repo has no automated tests, so I'd frame this as a gap I recognize: "I validated end-to-end manually and have a prioritized test list (dedup, HMAC, merge-handler, parser, upsert) ready to implement" — honest, not a claim of tests that don't exist.

**Deployment/production readiness.**
- *S:* A demo that actually runs vs slideware.
- *T:* Prove it works in the cloud.
- *A:* Per-service pipelines, migrations-as-Job, probes, HPA, monitoring; a real PR exercised the whole path.
- *R:* Reproducible deploy + clean teardown; readiness gaps (HTTPS, multi-AZ, tests) named openly.

### 15.7 "Explain This Project To…"

**…a recruiter.** "An AI bot that reviews code on GitHub automatically when a developer opens a pull request — catching security issues and bugs first. I built the AI, the backend, and all the AWS cloud infrastructure, with monitoring and automated quality checks. It shows I can ship an AI feature like a product."

**…a non-technical user.** "Imagine every time someone proposes a change to a codebase, a tireless assistant instantly reads it and leaves notes — 'this password shouldn't be here,' 'this could crash' — so the human reviewer starts from a checklist instead of a blank page."

**…a junior developer.** "Five small FastAPI services plus two Celery workers. A gateway checks the webhook is really from GitHub, a webhook service saves the PR and queues a job, an orchestrator pulls the diff and runs four LLM agents in parallel with LangGraph, a reviewer posts the comments, and a learner remembers recurring issues. Everything's containerized and deployed to Kubernetes with one CI pipeline per service."

**…a senior engineer.** (See §15.1 senior pitch + §15.2.) Emphasize: HMAC edge, 202-and-enqueue idempotency, Celery decoupling, LangGraph `Send`/reducer, GitHub App tokens, OIDC CI/CD, HPA, LLMOps (LangFuse + RAGAS), and the three known bugs with fixes.

**…a product manager.** "It reduces review latency and catches a class of issues humans skim past, as a first pass that never blocks the human. It's a validated prototype: proven on a real PR at ~a cent per review. To productize, I'd add tests, HTTPS/private networking, fix the known learning-loop bug, and ground inline comments — then pilot on a live repo."

**…a hiring manager.** "This is my evidence that I can own an LLM feature end to end — model design through cloud deployment, monitoring, and evaluation — and that I'm honest about engineering tradeoffs and my system's limitations. I can whiteboard any part and tell you exactly what I'd fix first."

**…an ML/AI engineer.** "Multi-agent inference via LangGraph: a conditional `Send` entry fans the diff to four role-specialized `gpt-4o-mini` agents with structured-JSON outputs, reconciled in a merge node (whose `operator.add` reducer is the source of a double-emit I can explain). LLMOps is real: LangFuse traces every call; a weekly RAGAS Job scores faithfulness/answer-relevancy and gates at 0.7. The 'learning' is prompt-conditioning from a per-repo frequency-ranked pattern store, not fine-tuning. I'd improve grounding (feed the diff as RAGAS context, ground line numbers), add structured outputs, and benchmark a larger model."

---

## 16. Glossary

| Term | Meaning in this project |
|---|---|
| **Agent** | One LLM node in the LangGraph graph with a role-specific prompt (static_analysis, security, style, architecture). |
| **ALB** | AWS Application Load Balancer; the internet-facing ingress in front of the gateway. |
| **Alembic** | Python DB-migration tool; `0001_initial` creates the schema. Autogenerate is off here. |
| **Answer relevancy** | A RAGAS metric: is the finding relevant to "what issues exist in this code?" Computed but not gated. |
| **asyncpg** | Async PostgreSQL driver used by SQLAlchemy (`postgresql+asyncpg://`). |
| **Celery** | Distributed task queue; tasks `analyze_pr` (queue `webhook`) and `trigger_learning` (queue `learning`). |
| **ConfigMap / Secret** | k8s objects (`app-config` / `app-secrets`) injected as env via `envFrom`. |
| **ECR** | Elastic Container Registry; 6 repos (5 services + evaluate). |
| **EKS** | Elastic Kubernetes Service; runs all workloads (k8s 1.32). |
| **ElastiCache** | Managed Redis (Celery broker/back-end). |
| **Faithfulness** | RAGAS metric: are findings grounded in the provided context? The CI gate (`< 0.7` fails). |
| **Fan-out (`Send`)** | LangGraph mechanism to dispatch the diff to all four agents in parallel from a conditional entry point. |
| **Finding** | One issue: `{file, line, severity, message, agent}`; persisted in `findings`. |
| **Gateway** | The HMAC-verification edge service (`:8000`). |
| **GitHub App** | The bot identity; mints short-lived installation tokens from a signed JWT. |
| **HMAC-SHA256** | Webhook signature scheme; verified constant-time against `X-Hub-Signature-256`. |
| **HPA** | HorizontalPodAutoscaler; scales the orchestrator 2→6 on 70% CPU. |
| **Idempotency** | App-level dedupe on `(repo_full_name, pr_number, head_sha)` in the webhook. |
| **Installation token** | Short-lived GitHub token (from the App JWT) used to read diffs / post reviews. |
| **LangFuse** | LLM observability; the OpenAI client is its drop-in so every call is traced. |
| **LangGraph** | Agent-orchestration framework; the graph = 4 agents + merge + END. |
| **Learner** | Service that upserts recurring findings into `patterns` on merge (`:8004`). |
| **Merge node** | Graph node that de-dupes findings by `(file,line,agent,message)`. |
| **OIDC** | Keyless GitHub-Actions→AWS auth (assume an IAM role; no stored keys). |
| **Orchestrator** | Service that fetches the diff and runs the agent graph (`:8002`). |
| **Pattern** | A recurring finding message stored per-repo with a `frequency` counter; conditions the style agent. |
| **RAGAS** | Offline LLM-evaluation framework used by the weekly `evaluate` Job. |
| **Reducer (`operator.add`)** | The LangGraph state-merge function on `findings` (list concatenation) — root of the double-emit bug. |
| **Reviewer** | Service that posts one consolidated GitHub review (`:8003`). |
| **Webhook (service)** | Persists the PR + enqueues analysis/learning (`:8001`). |
| **Webhook-worker / Learner-worker** | Celery workers consuming the `webhook` / `learning` queues. |

---

## 17. Risks, Gaps, And Improvement Roadmap

### Highest-risk code areas

1. **`services/learner/worker.py`** — does not register `trigger_learning`; the learner-worker likely can't run the queued task (`NotRegistered`), silently breaking the learning loop.
2. **`services/orchestrator/graph.py`** — the `operator.add` reducer + `merge_node` double-emit findings (also inflates learned-pattern frequencies and DB writes).
3. **`services/webhook/main.py`** — `scalar_one_or_none()` on the merge lookup raises on multi-commit PRs.
4. **`services/orchestrator/main.py`** — synchronous `build_graph().invoke()` blocks the async event loop; missing `raise_for_status` on the reviewer call hides downstream failures; no diff size/token cap.

### Missing tests

No tests at all. Top gaps: dedup/double-emit, HMAC, idempotency, merge-handler multi-commit, JSON parsing, `422` fallback, upsert frequency, Celery task registration (§12).

### Security concerns

HTTP-only ingress (`ingress.yaml`); public EKS endpoint + public subnets (`main.tf`); cluster-admin CI role; committed internal Redis hostname (`configmap.yaml`); extra public `gateway-lb` LoadBalancer; no rate limiting / request-size limits; diffs (proprietary code) leave to OpenAI/LangFuse (privacy disclosure).

### Performance concerns

Event-loop blocking in the orchestrator; per-request graph compilation; no DB indexes on hot lookups; doubled writes; new httpx clients per call; whole-diff prompts (token cost/latency).

### Maintainability concerns

Unpinned deps (no lockfiles); ORM/`Settings` duplicated across services; gitignored Terraform lockfile; stray `infra/k8s/kubectl`; dangling README `docs/*.png`; personal paths + region drift in `INSTRUCTIONS.md`.

### Documentation gaps

README images don't resolve; no `docs/`; no `.env.example` despite the gitignore allow-list; no `CONTRIBUTING`/local-dev guide; no architecture decision records beyond the README narrative.

### Suggested improvements — ordered by **impact**

1. Register `trigger_learning` in the learner worker (restores the learning loop).
2. Fix the double-emit (separate state key or dedupe-at-save) + add the regression test.
3. Fix the merge-handler multi-commit query.
4. Ground inline comment line numbers against the parsed diff (kills most `422` fallbacks).
5. Add the core test suite + pin dependencies + commit the TF lockfile.
6. Make the orchestrator graph call non-blocking; compile the graph once.
7. Harden infra for any real use: HTTPS/ACM, private subnets + NAT, multi-AZ DB, scoped CI IAM, secrets manager.
8. Add DB indexes; add Prometheus alert rules + worker metrics.

### Suggested improvements — ordered by **effort (low→high)**

- **Trivial:** delete `infra/k8s/kubectl`; commit `.terraform.lock.hcl`; add `.env.example`; fix README image paths; scrub personal paths/region in `INSTRUCTIONS.md`.
- **Small:** register the learner task; fix the merge query; fix the double-emit; pin deps.
- **Medium:** test suite; diff-grounded line numbers; async/threadpool graph call; DB indexes; alert rules.
- **Large:** infra hardening (HTTPS, private networking, multi-AZ, IRSA-scoped IAM); shared models package; larger-model quality benchmark; meaningful RAGAS dataset with diff-as-context.

---

## 18. Coverage Checklist

**Inventory method:** `git ls-files` (tracked files) + `git status --short` (untracked). Working tree was **clean** — there are **no untracked files**.

- **Total tracked files:** **64** — all accounted for below.
- **Total tracked folders (dirs containing tracked files):** **14** — `.`, `.github/workflows`, `db`, `db/migrations`, `db/migrations/versions`, `infra/k8s`, `infra/terraform`, `monitoring`, `scripts`, `services/gateway`, `services/learner`, `services/orchestrator`, `services/reviewer`, `services/webhook` (plus implied parents `.github`, `infra`, `services`). All explained in §9.
- **Notable untracked files:** **none** (clean tree). The real secret overlay (`infra/k8s/secret.local.yaml`) is gitignored and **not present**; no secret values were read or copied.
- **Binary/empty handled per rules:** `ai-code-review-pr-screenshot.pdf` (binary, documented by role) and `infra/k8s/kubectl` (0-byte stray, documented as such).

| # | Tracked file | Coverage | Where |
|---|---|---|---|
| 1 | `.gitignore` | Deep dive | §10.8 |
| 2 | `INSTRUCTIONS.md` | Deep dive (phase-level) | §10.8, §3, §13 |
| 3 | `LICENSE` | Deep dive | §10.8 |
| 4 | `README.md` | Deep dive | §10.8 (+ basis throughout) |
| 5 | `ai-code-review-pr-screenshot.pdf` | High-level (binary, by rule) | §10.8 |
| 6 | `.github/workflows/service-ci.yml` | Deep dive | §10.7 |
| 7 | `.github/workflows/gateway.yml` | Deep dive (group) | §10.7 |
| 8 | `.github/workflows/webhook.yml` | Deep dive (group) | §10.7 |
| 9 | `.github/workflows/orchestrator.yml` | Deep dive (group) | §10.7 |
| 10 | `.github/workflows/reviewer.yml` | Deep dive (group) | §10.7 |
| 11 | `.github/workflows/learner.yml` | Deep dive (group) | §10.7 |
| 12 | `.github/workflows/evaluate.yml` | Deep dive | §10.7 |
| 13 | `db/alembic.ini` | Deep dive | §10.2 |
| 14 | `db/migrations/env.py` | Deep dive | §10.2 |
| 15 | `db/migrations/versions/0001_initial.py` | Deep dive | §10.2, §7.1 |
| 16 | `infra/k8s/configmap.yaml` | Deep dive | §10.6 |
| 17 | `infra/k8s/secret.yaml` | Deep dive | §10.6 |
| 18 | `infra/k8s/gateway.yaml` | Deep dive | §10.6 |
| 19 | `infra/k8s/webhook.yaml` | Deep dive (group) | §10.6 |
| 20 | `infra/k8s/orchestrator.yaml` | Deep dive (group) | §10.6 |
| 21 | `infra/k8s/reviewer.yaml` | Deep dive (group) | §10.6 |
| 22 | `infra/k8s/learner.yaml` | Deep dive (group) | §10.6 |
| 23 | `infra/k8s/webhook-worker.yaml` | Deep dive | §10.6 |
| 24 | `infra/k8s/learner-worker.yaml` | Deep dive | §10.6 |
| 25 | `infra/k8s/hpa.yaml` | Deep dive | §10.6 |
| 26 | `infra/k8s/ingress.yaml` | Deep dive | §10.6 |
| 27 | `infra/k8s/migration-job.yaml` | Deep dive | §10.6 |
| 28 | `infra/k8s/evaluate-job.yaml` | Deep dive | §10.6 |
| 29 | `infra/k8s/kubectl` | High-level (0-byte stray, by rule) | §10.6 |
| 30 | `infra/terraform/main.tf` | Deep dive | §10.5 |
| 31 | `infra/terraform/outputs.tf` | Deep dive | §10.5 |
| 32 | `infra/terraform/variables.tf` | Deep dive | §10.5 |
| 33 | `monitoring/grafana-dashboard.json` | Panel-level (generated/config, by rule) | §10.4 |
| 34 | `monitoring/prometheus.yml` | Deep dive | §10.4 |
| 35 | `scripts/Dockerfile` | Deep dive | §10.3 |
| 36 | `scripts/evaluate.py` | Deep dive | §10.3 |
| 37 | `services/gateway/Dockerfile` | Deep dive (group) | §10.1 |
| 38 | `services/gateway/deploy.txt` | Deep dive (group) | §10.1 |
| 39 | `services/gateway/main.py` | Deep dive | §10.1 |
| 40 | `services/gateway/models.py` | Deep dive | §10.1 |
| 41 | `services/gateway/requirements.txt` | Deep dive (group) | §10.1 |
| 42 | `services/learner/Dockerfile` | Deep dive (group) | §10.1 |
| 43 | `services/learner/deploy.txt` | Deep dive (group) | §10.1 |
| 44 | `services/learner/main.py` | Deep dive | §10.1 |
| 45 | `services/learner/models.py` | Deep dive | §10.1 |
| 46 | `services/learner/requirements.txt` | Deep dive (group) | §10.1 |
| 47 | `services/learner/worker.py` | Deep dive | §10.1 |
| 48 | `services/orchestrator/Dockerfile` | Deep dive (group) | §10.1 |
| 49 | `services/orchestrator/deploy.txt` | Deep dive (group) | §10.1 |
| 50 | `services/orchestrator/graph.py` | Deep dive ⭐ | §10.1 |
| 51 | `services/orchestrator/main.py` | Deep dive | §10.1 |
| 52 | `services/orchestrator/models.py` | Deep dive | §10.1 |
| 53 | `services/orchestrator/requirements.txt` | Deep dive (group) | §10.1 |
| 54 | `services/reviewer/Dockerfile` | Deep dive (group) | §10.1 |
| 55 | `services/reviewer/deploy.txt` | Deep dive (group) | §10.1 |
| 56 | `services/reviewer/main.py` | Deep dive | §10.1 |
| 57 | `services/reviewer/models.py` | Deep dive | §10.1 |
| 58 | `services/reviewer/requirements.txt` | Deep dive (group) | §10.1 |
| 59 | `services/webhook/Dockerfile` | Deep dive (group) | §10.1 |
| 60 | `services/webhook/deploy.txt` | Deep dive (group) | §10.1 |
| 61 | `services/webhook/main.py` | Deep dive | §10.1 |
| 62 | `services/webhook/models.py` | Deep dive | §10.1 |
| 63 | `services/webhook/requirements.txt` | Deep dive (group) | §10.1 |
| 64 | `services/webhook/worker.py` | Deep dive | §10.1 |

**Files only covered at high level (with reason):**
- `ai-code-review-pr-screenshot.pdf` — binary PDF; per the binary-file rule, documented by path/role/tracked-status, not decoded.
- `infra/k8s/kubectl` — 0-byte stray file; no content to analyze; flagged for deletion.
- `monitoring/grafana-dashboard.json` — first-party generated/config artifact; documented at panel/structure level rather than line-by-line JSON.

**Files skipped entirely:** none.

**"Deep dive (group)" note:** the six Dockerfiles, five `deploy.txt`, five per-service workflow triggers, and five near-identical service Deployments are documented once in full with an explicit per-file delta table — every individual file's specifics (port, name, extra resources) are stated, so each is fully accounted for.

**Limitations of this analysis:**
- Static analysis only — nothing was executed; the three suspected bugs (learner `NotRegistered`, double-emit, multi-commit merge) are reasoned from code/config and the LangGraph/Celery semantics, not reproduced live. Each is labeled as such.
- Cloud-runtime behavior (whether the Grafana `app` label resolves, whether S3 is used, actual scrape path) cannot be confirmed from the repo and is marked *"Unclear from repo evidence."*
- `INSTRUCTIONS.md` and `grafana-dashboard.json` are summarized at section/panel granularity rather than line-by-line, given their length and config nature.
- No secret values exist in tracked files; none were copied. Variable names only, values shown as `[REDACTED]`.

---

*End of `ai-code-reviewer_ALLINFO.md`.*








