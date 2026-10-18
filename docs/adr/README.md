# Architecture Decision Records

Each ADR records one decision that is hard to reverse or that affects cost, security or the boundaries between components (see [constitution §1.2](../../specs/00-constitution.md)).
Every ADR is checked against the six **AWS Well-Architected** pillars, and it ends with **SAA-C03 exam notes** (the project doubles as preparation for AWS Solutions Architect Associate).

Statuses: `Proposed → Accepted → Superseded | Deprecated`. All ADRs below are **Proposed** and get accepted together with [spec 01](../../specs/01-mvp.md).

| # | Title | Status | Decision in one line |
|---|---|---|---|
| [0001](0001-dataset.md) | Training dataset | Proposed | IEEE-CIS Fraud Detection, not PaySim |
| [0002](0002-event-stream.md) | Event stream on AWS | Proposed | Kinesis Data Streams, provisioned 1 shard, ephemeral |
| [0003](0003-velocity-feature-store.md) | Velocity feature store | Proposed | DynamoDB on-demand, per-event items + TTL |
| [0004](0004-model-serving.md) | Model serving | Proposed | ECS Fargate (ARM64, public IP, no ALB/NAT), ephemeral |
| [0005](0005-infrastructure-as-code.md) | Infrastructure as Code | Proposed | AWS CDK in Python + cdk-nag |

Prices quoted in ADRs are approximate on-demand list prices as known when the ADR was written, and they differ by region. Re-check them in the [AWS Pricing Calculator](https://calculator.aws/) before relying on them.

New ADRs: copy [`template.md`](template.md).
