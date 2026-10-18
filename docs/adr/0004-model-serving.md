# 0004 — Model serving: ECS Fargate (ARM64, public IP, no ALB/NAT), ephemeral

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-18 |
| **Deciders** | Kacper Celejewski |
| **Related** | [spec 01 §5, §8](../../specs/01-mvp.md), [ADR 0002](0002-event-stream.md), [ADR 0003](0003-velocity-feature-store.md) |

## Context

Two workloads run the same `ScoreTransaction` use case:

1. **Synchronous API** `POST /score`, with a server-side p95 target < 100 ms including DynamoDB round trips.
2. **Stream consumer** reading Kinesis continuously and checkpointing in DynamoDB (ADR 0002).

Both load the same LightGBM model bundle (a ~10–50 MB model; the image is ~300–500 MB with numpy/lightgbm). Constraints: **< $5/month**, ephemeral environment, the same container as in local docker-compose, and no NAT Gateway or ALB (constitution §6).

## Decision

**Amazon ECS on AWS Fargate**, in the ephemeral `RuntimeStack`:

- One ECS service, desired count **1**, one task with **two containers from one image**: `api` (uvicorn/FastAPI) and `consumer` (stream loop). Different commands, shared task role.
- Task size **0.5 vCPU / 1 GB**, **ARM64 (Graviton)**, Linux.
- **Public subnet + `assignPublicIp`**. No NAT Gateway (image pull from ECR, DynamoDB, Kinesis, S3 and CloudWatch all go over the public endpoints with TLS + IAM). **No ALB.** The demo calls the task's public IP directly.
- Security group ingress: TCP 8000 **only from `operator_cidr`** (a CDK context parameter). No `0.0.0.0/0`.
- The model bundle is **baked into the image** at build time (spec 01 Q5, a candidate for ADR 0006).
- Logs go to CloudWatch Logs with 1-day retention. A container health check calls `/health`.

## Alternatives considered

Approximate `us-east-1` prices. "Ephemeral" = ~20 h/month.

| Option | Always-on / month | Ephemeral 20 h | Latency | Fit for stream consumer | Why not |
|---|---|---|---|---|---|
| **ECS Fargate ARM64, 0.5 vCPU / 1 GB, public IP** | ~$14 compute + ~$3.6 public IPv4 | **~$0.50** | Warm process, model in memory, no cold starts | A long-running loop is natural | **Chosen** |
| Fargate + ALB (+ NAT for private subnets) | +~$16 ALB, +~$32 NAT per AZ | +~$1.5 | Same | Same | The production-shaped design, but it breaks the budget if left on. Documented as the "production delta". |
| **Lambda container image** + Function URL; Kinesis event source mapping for the stream | **≈ $0** within the always-free tier (1M requests, 400k GB-s) | ≈ $0 | **Cold start** of a 300–500 MB image with numpy/lightgbm: about 1–3 s (provisioned concurrency removes it but costs money) | Event source mapping handles polling, batching and checkpointing for you | A **close second**, and cheaper at idle. Rejected for the MVP because (a) ephemeral deployment already removes idle cost, (b) cold starts break p95 for a sparse demo workload, and (c) local docker-compose and cloud would run different runtime models. **First choice if the environment ever has to be always-on.** |
| SageMaker real-time endpoint (`ml.t2.medium`/`ml.m5.large`) | ~$40–85 | ~$1–2, plus 5–10 min endpoint creation | Good | **None.** The endpoint only does inference, so velocity features and stream consumption still need a separate compute service. | Two services instead of one, budget, and it would give the model server ownership of the feature logic. |
| SageMaker Serverless Inference | Pay per ms + cold starts | ≈ cents | Cold starts | None (same issue) | Same split-brain problem. Memory ≤ 6 GB. |
| EC2 `t4g.micro` (free-tier eligible on some accounts) | ~$6 or free tier | ~$0.20 | Good | Yes | Patching, AMIs and process supervision by hand, which is less to show and more to maintain. A good exam contrast, though. |

## Consequences

**Positive**
- The same OCI image and the same process model locally and on AWS.
- No cold starts. The model loads once at container start, and `/ready` gates traffic.
- ECS concepts (task vs execution role, task definitions, awsvpc networking, service desired count) are directly exam-relevant.
- ARM64 is ~20% cheaper per vCPU-hour and more energy-efficient.

**Negative**
- **No stable endpoint.** The public IP changes on each deployment. The demo script reads it from ECS (`DescribeTasks`). No TLS on the raw IP: acceptable for an IP-restricted demo, not for production.
- **Single task = single point of failure.** Acceptable for the MVP (no HA, spec 01 §3).
- The consumer and API share CPU. Under a heavy stream replay, API latency will rise. Measure it (AC-3.3) and split into two services if needed.
- **Public IPv4 addresses are billed** (~$0.005/h). Negligible when ephemeral, but it shows why IPv6 or private subnets matter at scale.
- The image must be built for `linux/arm64`. Building on an Apple Silicon Mac is native, but CI runners on x86 need buildx/QEMU or ARM runners.

**Revisit if:** the environment has to be always-on (→ Lambda), HA/TLS is required (→ ALB + ACM certificate + 2 tasks across AZs + private subnets with gateway endpoints), or GPU or multi-model serving is required (→ SageMaker).

## Well-Architected lens

| Pillar | Assessment |
|---|---|
| Operational Excellence | Everything in CDK. Structured JSON logs carrying `model_version`. Container health checks. `/ready` reflects whether the model is loaded and the store is reachable. One image for both roles keeps deployments atomic. |
| Security | **Task role** (app permissions: one stream, one table, one bucket prefix) is separate from the **execution role** (ECR pull, logs). IP allowlist. Read-only root filesystem and a non-root user in the container. No secrets needed. |
| Reliability | Deliberately a single task (MVP). ECS restarts failed tasks. The stream checkpoint lets the consumer resume. The production delta is ≥ 2 tasks across AZs behind an ALB. |
| Performance Efficiency | Warm in-memory model. TreeSHAP via LightGBM native `pred_contrib` (no extra library). Size is checked against AC-3.3 measurements, not guessed. |
| Cost Optimization | ARM64, smallest viable size, ephemeral, no ALB/NAT. Cost per demo hour is ~$0.02 for compute + IP. |
| Sustainability | Graviton (better performance per watt). Runs only during demos. Shared image reduces build and storage. |

## SAA-C03 exam notes

- **ECS launch types:** EC2 (you manage instances, can use Spot/RI, cheaper at high utilisation) vs **Fargate** (serverless, per vCPU-second and GB-second). **EKS** = Kubernetes. Fargate also works with EKS.
- **Task role vs task execution role:** the execution role lets the ECS agent pull images and write logs. The task role is what your code uses for AWS API calls. This is a frequent trick question.
- **awsvpc network mode:** each Fargate task gets its own ENI and security group.
- Private-subnet Fargate needs **NAT Gateway** *or* VPC endpoints: interface endpoints for ECR (`ecr.api`, `ecr.dkr`), CloudWatch Logs and Kinesis, plus a **gateway endpoint for S3** (ECR layers live in S3) and DynamoDB.
- **Lambda limits:** 15 min timeout, up to 10,240 MB memory, container images up to 10 GB, `/tmp` up to 10 GB. Cold-start mitigations: provisioned concurrency, SnapStart (Java/Python/.NET), smaller packages.
- **SageMaker inference options:** real-time (persistent endpoint), serverless (sporadic traffic, cold starts), asynchronous (large payloads, queue-based, can scale to zero), batch transform (offline scoring).
- **ALB vs NLB:** ALB = L7 (HTTP routing, host/path rules). NLB = L4 (TCP/UDP, static IP/Elastic IP, extreme throughput). Both are billed hourly + per LCU/NLCU.
