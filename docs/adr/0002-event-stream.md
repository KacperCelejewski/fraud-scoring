# 0002 — Event stream on AWS: Kinesis Data Streams (provisioned, 1 shard, ephemeral)

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-18 |
| **Deciders** | Kacper Celejewski |
| **Related** | [spec 01 §5](../../specs/01-mvp.md), [ADR 0004](0004-model-serving.md), constitution §6 (cost) |

## Context

Transaction events (`TransactionEvent v1`) flow from a producer (the Python generator now, a Java/Spring payment service later) to the scoring consumer. Requirements:

- **Per-card ordering.** Velocity features are read-before-write per card (spec 01 §5.1).
- **Replay.** The same window should be re-run through the stream to compare model versions, and a second consumer (shadow challenger, archiver) may read the same events independently.
- **At-least-once delivery is acceptable.** Idempotency is handled in the velocity store (ADR 0003).
- **Volume:** demo-scale, below 100 events/s.
- **Budget: < $5/month in total**, with infrastructure up only during demo sessions (estimate: ≤ 20 h/month).
- Local parity: LocalStack must emulate it.

## Decision

**Amazon Kinesis Data Streams, provisioned capacity mode, 1 shard, 24 h retention, deployed only in the ephemeral `RuntimeStack`.**

- Partition key = `card_id`, so ordering holds per card within the shard.
- Server-side encryption with the AWS-managed KMS key (`aws/kinesis`).
- Consumer: a single polling consumer with its checkpoint (sequence number) in DynamoDB. No enhanced fan-out.
- Producer uses `PutRecords` batches (≤ 500 records) with retry of failed records.

## Alternatives considered

Approximate `us-east-1` list prices (Frankfurt is somewhat higher). "Ephemeral" means ~20 h/month.

| Option | Always-on / month | Ephemeral 20 h / month | Ordering | Replay / multi-consumer | Why not |
|---|---|---|---|---|---|
| **Kinesis provisioned, 1 shard** | ~$11 ($0.015/shard-h) + PUT units (~$0.014/M) | **~$0.30** | Per partition key (shard) | Yes: retention 24 h–365 d, many consumers | **Chosen** |
| Kinesis on-demand | ~$29 ($0.04/stream-h) + $/GB in/out | ~$0.80 | Per partition key | Yes | ~3× the provisioned cost at this volume. Autoscaling is not needed for 1 shard of traffic. |
| **SQS FIFO** (message group = `card_id`) | ≈ $0 (1M requests/month free) | ≈ $0 | Per message group | **No replay** (messages are deleted after processing). Fan-out needs SNS. | Cheapest and technically enough for scoring, but loses replay and independent consumers, which are central to shadow evaluation. A strong fallback. |
| SQS Standard | ≈ $0 | ≈ $0 | None (best-effort) | No | Velocity read-before-write per card needs ordering. |
| **MSK provisioned** (2 × `kafka.t3.small`) | ~$65–75 + storage | Cluster creation takes 15–30+ min, so not practical | Per partition | Yes | 10–15× the whole budget. Needs a VPC with private connectivity. Kacper already uses Kafka daily, so little learning value. |
| MSK Serverless | ~$550 (≈ $0.75/cluster-h + partitions) | ~$15+ | Per partition | Yes | Far above budget. |
| EventBridge | ≈ $1/M events | ≈ $0 | None | Archive & replay exists but is coarse | It is an event *router*, not an ordered stream. |

## Consequences

**Positive**
- Shows a real streaming design (partition keys, shard limits, checkpointing, at-least-once semantics, replay), all of which carry over to the Kafka world Kacper knows.
- Costs cents per month when deployed only for demos.
- LocalStack supports Kinesis, so local and cloud use the same boto3 code paths.

**Negative**
- **Ephemeral means no persistent history.** Destroying the stack deletes the stream and its retained data. This is acceptable because the source of truth is the dataset plus the S3 decision log.
- 1 shard = 1 MB/s or 1,000 records/s in, 2 MB/s out shared across consumers (5 `GetRecords` calls/s per shard). This is enough for the MVP. Scaling means resharding plus a multi-shard consumer (spec 01 §11).
- The hand-written single-shard consumer does not do lease balancing. Running two consumer tasks would double-process. Mitigation: ECS desired count = 1, and processing is idempotent anyway.
- Always-on would cost ~$11/month for the stream alone, so "ephemeral" is a hard requirement, not a nice-to-have.

**Revisit if:** the environment must be always-on (switch to SQS FIFO and accept losing replay), or traffic needs more than one shard.

## Well-Architected lens

| Pillar | Assessment |
|---|---|
| Operational Excellence | The stream is defined in CDK and deployed/destroyed per session. CloudWatch metrics to watch: `IncomingRecords`, `GetRecords.IteratorAgeMilliseconds` (consumer lag), `WriteProvisionedThroughputExceeded`. |
| Security | SSE with KMS at rest, TLS in transit. Producer and consumer IAM roles get only `kinesis:PutRecord(s)` / `kinesis:GetRecords`, `GetShardIterator`, `DescribeStream` on **this** stream ARN. |
| Reliability | Data is replicated across 3 AZs by the service. Retention (24 h) allows replay after a consumer crash, using the checkpoint in DynamoDB. The producer retries the partial failures returned by `PutRecords`. |
| Performance Efficiency | Partition key `card_id` spreads load (high cardinality, so no hot shard at 1 shard anyway). Batching on both sides. |
| Cost Optimization | Provisioned mode is ~3× cheaper than on-demand at steady low volume. Ephemeral deployment. No enhanced fan-out (charged per consumer-shard-hour). |
| Sustainability | Resources exist only while used. Smallest possible capacity unit. |

## SAA-C03 exam notes

- **Kinesis Data Streams vs SQS vs SNS vs MSK:** KDS = ordered per shard, replayable, multiple consumers, real-time analytics. SQS = decoupling/work queue, a message is consumed by one consumer, no replay. SNS = pub/sub fan-out (SNS → multiple SQS queues is the classic fan-out). MSK = managed Kafka, used when you need Kafka APIs or are migrating.
- **Shard limits:** 1 MB/s or 1,000 records/s write, 2 MB/s read per shard. `ProvisionedThroughputExceededException` means a hot partition key or too few shards. Fix with better key distribution or by splitting shards.
- **Retention:** default 24 h, extendable up to 365 days (extra charge).
- **Enhanced fan-out:** dedicated 2 MB/s per consumer per shard, push via HTTP/2 (`SubscribeToShard`). Use it when many consumers would hit the shared 2 MB/s limit.
- **Kinesis Data Firehose (Amazon Data Firehose)** = near-real-time *delivery* to S3/Redshift/OpenSearch with buffering. It does not keep data for replay. It would be the managed way to land the decision log in S3.
- **SQS FIFO:** ordering per `MessageGroupId`, exactly-once processing via deduplication ID (5-minute window), 300 msg/s per API action without batching / 3,000 with batching (higher in high-throughput mode). **Visibility timeout** and **DLQ** are frequent exam topics.
- **Lambda + Kinesis:** event source mapping with batch size, `ParallelizationFactor` (up to 10 concurrent batches per shard), bisect-on-error, and on-failure destination.
