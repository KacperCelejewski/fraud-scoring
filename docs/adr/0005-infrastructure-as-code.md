# 0005 — Infrastructure as Code: AWS CDK in Python (+ cdk-nag)

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-18 |
| **Deciders** | Kacper Celejewski |
| **Related** | constitution §2, §5, §6; [spec 01 §5.2](../../specs/01-mvp.md); ADRs [0002](0002-event-stream.md)–[0004](0004-model-serving.md) |

## Context

All AWS resources must be reproducible from code, cheap to **create and destroy repeatedly** (the ephemeral runtime stack), reviewed for security, and written in a language Kacper is moving towards (Python). The project is AWS-only. Being able to read CloudFormation and IAM is also useful for the SAA exam.

## Decision

**AWS CDK v2 in Python**, in `infra/` (its own `pyproject.toml`, managed by uv):

| Stack | Lifetime | Contents |
|---|---|---|
| `PersistentStack` | Deployed once | S3 bucket (decision log, model bundles; Block Public Access, SSE-S3, lifecycle → expire after 30 days), ECR repository (lifecycle: keep last 3 images), **AWS Budgets** ($3 forecast / $5 actual → email) |
| `RuntimeStack` | **Deployed per demo session, then destroyed** | VPC (public subnets only, 2 AZs, `natGateways: 0`), Kinesis stream, DynamoDB table(s) (`RemovalPolicy.DESTROY`), ECS cluster/service/task definition, log group (1-day retention), IAM roles |

- **cdk-nag** `AwsSolutionsChecks` runs on every `cdk synth`. Findings are either fixed or suppressed with a written reason (e.g. "AwsSolutions-EC23: ingress restricted to operator CIDR, demo only").
- Tags `project=fraud-scoring`, `env=<stage>` are set at the app level (constitution §6).
- `operator_cidr` and the model version are passed through CDK context. No hard-coded account IDs (`env` comes from the CLI profile).
- Snapshot/assertion tests for stacks use `aws_cdk.assertions` in pytest (e.g. "no NAT gateway", "no `0.0.0.0/0` ingress", "all tables have `DESTROY` removal policy").

## Alternatives considered

| Option | Pros | Cons | Why not |
|---|---|---|---|
| **AWS CDK (Python)** | Same language as the app. High-level L2 constructs (`grant_read()` produces least-privilege IAM). State managed by CloudFormation (no state bucket). Ideal for create/destroy cycles. `assertions` module for tests. cdk-nag. | Abstractions hide what gets created (read `cdk synth` output). CloudFormation can be slow and has stuck-rollback states (`UPDATE_ROLLBACK_FAILED`). AWS-only. Needs `cdk bootstrap` (creates a staging bucket + ECR repo, near-zero cost). | **Chosen** |
| **Terraform (HCL)** | Industry standard, multi-cloud, `plan` diffs are very clear, a huge module ecosystem. A strong CV keyword. | You manage state yourself (S3 backend + locking). HCL is another language. Least-privilege IAM is more manual. Less relevant to the SAA exam (CloudFormation is what gets tested). | A good choice in a multi-cloud team. Here CDK's language unity and CloudFormation exam relevance win. Kacper may meet Terraform at work, so the trade-off is explicit. |
| CloudFormation / SAM YAML | No abstraction, exactly what the exam tests. SAM is great for Lambda. | Verbose. Loops and conditions are awkward. No real tests. | Too much YAML for an ECS+Kinesis+DynamoDB stack. CDK synthesises CloudFormation anyway, so the YAML can still be read. |
| Pulumi (Python) | Python, multi-cloud | Separate state service/backend. Smaller AWS-specific construct ecosystem. Less common in job ads. | No strong advantage over CDK for AWS-only work. |
| ClickOps / CLI scripts | Fastest to start | Not reproducible, easy to leave billable resources behind | Violates the constitution. |

## Consequences

**Positive**
- `cdk deploy RuntimeStack` / `cdk destroy RuntimeStack` gives a full environment lifecycle in one command each, which is essential for the < $5 budget.
- IAM policies come from `grant_*` calls, so least privilege is the default rather than a chore.
- Infrastructure is tested in pytest like application code. Policy rules (no NAT, no open ingress) are executable.

**Negative**
- Destroying a stack fails if a bucket is non-empty or a resource has `RETAIN`. That is why S3 lives in `PersistentStack` and runtime resources use `DESTROY`.
- CloudFormation drift (manual console changes) is not visible unless drift detection is run. Rule: no console changes.
- The CDK version has to be pinned (`aws-cdk-lib` + CLI) to avoid synth differences.
- Recruiters may look for Terraform. The README can point out that the decision is documented here.

**Revisit if:** a second cloud provider joins the project, or the team (future work) standardises on Terraform.

## Well-Architected lens

| Pillar | Assessment |
|---|---|
| Operational Excellence | All infrastructure is versioned, reviewed and tested. Deployments are repeatable, and changes are small and reversible (the Well-Architected "IaC" and "make frequent, small, reversible changes" practices). |
| Security | cdk-nag on every synth. `grant_*` gives least privilege. Block Public Access + encryption by default. No credentials in code (CLI profile / SSO). |
| Reliability | CloudFormation rolls back failed deployments automatically. The environment can be recreated from scratch in minutes, so recovery means redeploying. |
| Performance Efficiency | Instance types and sizes are parameters, easy to tune after AC-3.3 measurements. |
| Cost Optimization | Persistent/runtime stack split plus `RemovalPolicy.DESTROY` keeps idle cost near $0. Budgets alarm is part of the code. Cost-allocation tags are applied everywhere. |
| Sustainability | Resources exist only while used. Lifecycle rules expire old images and logs. |

## SAA-C03 exam notes

- **CloudFormation basics:** stacks, change sets (preview changes), **StackSets** (multi-account/region), drift detection, `DeletionPolicy: Retain | Snapshot | Delete`, rollback on failure, nested stacks, outputs/exports for cross-stack references.
- **CDK** synthesises CloudFormation. Constructs: **L1** (`Cfn*`, 1:1 with CloudFormation), **L2** (opinionated with defaults and `grant_*`), **L3** (patterns, e.g. `ApplicationLoadBalancedFargateService`). `cdk bootstrap` creates the `CDKToolkit` stack.
- **AWS Budgets** (alerts on actual/forecast cost, can trigger actions) vs **Cost Explorer** (analysis) vs **Cost and Usage Report** (raw detail). **Cost allocation tags** must be activated in Billing before they appear.
- **Elastic Beanstalk** = managed PaaS deployment of an app, not general IaC. **AWS Config** = resource compliance rules, a counterpart to cdk-nag at runtime.
- **Service Control Policies (Organizations)** can block expensive services account-wide, an alternative guardrail to Budgets.
