# 00 — Constitution

| | |
|---|---|
| **Status** | Draft (awaiting approval by Kacper) |
| **Owner** | Kacper Celejewski |
| **Created** | 2026-10-18 |
| **Applies to** | Every spec, ADR and line of code in this repository |

These are the project's invariants. A spec or ADR may **not** contradict them.
Changing the constitution needs its own ADR that explains why.

---

## 1. Spec-driven process

### 1.1 Lifecycle

```
Draft ──► Approved ──► Implemented ──► Verified
  ▲          │
  └──────────┘  (changes requested)
```

| Status | Meaning | Who moves it |
|---|---|---|
| **Draft** | Being written or discussed. Anything can change. | Anyone |
| **Approved** | Scope and acceptance criteria are frozen. Implementation may start. | **Kacper only** |
| **Implemented** | Code merged and every acceptance criterion has a test or a reproducible check. | Implementer |
| **Verified** | Acceptance criteria checked end to end (locally and, where relevant, on AWS), with the evidence linked in the spec. | Kacper |

Rules:

- **No implementation without an Approved spec.** Exploratory notebooks are allowed in `notebooks/` but are never merged into `src/`.
- If an Approved spec needs a material change (scope, API, metrics), it goes back to **Draft** and is re-approved.
- Every spec starts with a header table: status, owner, dates, related ADRs.
- Acceptance criteria are **checklists** (`- [ ]`). Each item can be verified by a test, a command, or an artefact such as a report or screenshot.

### 1.2 Architecture Decision Records

- **Every architectural decision is an ADR** in `docs/adr/NNNN-kebab-title.md`, using [the template](../docs/adr/template.md): Context / Decision / Alternatives / Consequences.
- Every ADR includes an **AWS Well-Architected** section covering the six pillars: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.
- ADR statuses: `Proposed → Accepted → (Superseded by NNNN | Deprecated)`. Accepted ADRs are never edited in substance. They are superseded by a new ADR.
- "Architectural" means it is hard to reverse, or it affects cost, security, or the boundaries between components. Choosing a library for one function is not architectural.

## 2. Technology stack

| Concern | Invariant |
|---|---|
| Language | **Python ≥ 3.12** |
| Packaging / env | **uv** (`pyproject.toml` + committed `uv.lock`) |
| Lint / format | **ruff** (lint + format), zero warnings in CI |
| Typing | **mypy `--strict`** on `src/` (pyright allowed in the editor; mypy is the gate) |
| Tests | **pytest** (+ `hypothesis` for property-based tests of time-window logic) |
| Data models at boundaries | **Pydantic v2** (API DTOs, event contracts, settings) |
| ML | LightGBM. Experiment tracking in MLflow. |
| API | FastAPI |
| IaC | **AWS CDK in Python**, checked by `cdk-nag` |
| Local env | docker-compose (LocalStack for Kinesis/DynamoDB/S3, MLflow) |

## 3. Architecture rules

### 3.1 Layers

```
src/fraud_scoring/
├── domain/          # pure Python: entities, value objects (Money), feature definitions,
│                    # decision policy, reason codes. NO I/O, NO boto3/fastapi/lightgbm imports.
├── application/     # use cases (ScoreTransaction, TrainModel, EvaluateModel) and ports (Protocols)
└── infrastructure/  # adapters: DynamoDB, Kinesis, S3, MLflow, FastAPI, LightGBM model loader
```

- Dependencies point **inwards only**: `infrastructure → application → domain`. This is enforced with `import-linter` in CI.
- Ports are `typing.Protocol`s defined in `application`. Adapters implement them in `infrastructure`.
- **One implementation of feature logic.** Velocity features are defined once in `domain` and used by both the offline (training) and online (serving) paths. Training/serving skew is a bug, not a tuning problem.

### 3.2 Money and time

- **Amounts are integers in minor units** (`amount_minor: int`) plus an ISO 4217 `currency`. `float` is never used for money in the domain. Derived model features (e.g. `log(amount)`) may be floats.
- All timestamps are **timezone-aware UTC**. Every event has an **event time** (`event_ts`), and that is the time used for windows, splits and TTLs. Processing time is only for metrics.
- Cost figures in specs and reports are in **PLN**. FX assumptions are explicit configuration, never hard-coded.

## 4. Data and ML integrity

- **No future leakage.** Train/validation/test splits are **by time** (out-of-time). Random K-fold on transactions is forbidden for model selection.
- **Point-in-time features only.** A feature for a transaction at time *t* may use only events with `event_ts < t`. Whole-dataset aggregates (e.g. frequency encodings computed over all rows) are forbidden.
- **Delayed labels are modelled, not ignored.** A label is used for training only if it would have been *known* at the training cutoff (see `specs/01-mvp.md` §4.3). Immature transactions are never treated as legitimate.
- **Every reported metric is out-of-time** and records the dataset version, split boundaries, code commit, and MLflow run ID.
- **Reproducibility:** fixed seeds, pinned dependencies, and a data-prep pipeline that can be re-run from the raw download.
- **Raw data is never committed** (dataset licences, size). The repo contains download instructions and checksums only.

## 5. Security

- **No secrets in the repo**, ever. Locally: `.env` (git-ignored) with only an `.env.example` committed. On AWS: IAM roles (task role); Secrets Manager / SSM Parameter Store if a secret is ever needed.
- **No real cardholder data.** Card identifiers are pseudonymous tokens. A PAN-like value never appears in events, logs or features.
- **Least privilege:** each ECS task role gets access only to its own resources (one stream, one table, one bucket prefix). No `*` actions in IAM policies, and `cdk-nag` findings are fixed or suppressed with a written reason.
- Network ingress to demo endpoints is limited to the operator's IP (CIDR parameter), never `0.0.0.0/0`.

## 6. Cost guardrails (hobby budget)

- **Target: < $5 / month** in AWS spend.
- Runtime infrastructure is **ephemeral**: deployed for a demo or test session and destroyed afterwards (`cdk destroy`). Only the near-free parts persist (S3 bucket, ECR repo, CDK bootstrap).
- Not allowed without a new ADR: NAT Gateway, Application Load Balancer, MSK, ElastiCache, always-on SageMaker endpoints, interface VPC endpoints.
- An **AWS Budgets** alert at $3 (forecast) and $5 (actual) is part of the infrastructure.
- Every resource is tagged `project=fraud-scoring` and `env=<name>` for Cost Explorer.

## 7. Engineering conventions

- **Commits:** Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`), written in English.
- **Language:** all repository content (code, docs, specs, ADRs) is in English.
- **Definition of Done for a change:** `ruff check`, `ruff format --check`, `mypy --strict`, `pytest` and `lint-imports` all pass. New behaviour has tests. The relevant spec/ADR is updated.
- **Observability baseline:** structured JSON logs that carry `transaction_id`, `model_version` and latency. No PII in logs.
