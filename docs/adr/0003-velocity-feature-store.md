# 0003 — Velocity feature store: DynamoDB (on-demand) with per-event items and TTL

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-18 |
| **Deciders** | Kacper Celejewski |
| **Related** | [spec 01 §4.2, §5.1](../../specs/01-mvp.md), [ADR 0002](0002-event-stream.md), [ADR 0004](0004-model-serving.md) |

## Context

Online scoring needs per-card velocity features over sliding windows of 10 min, 1 h and 24 h (counts, amount sums, time since the previous transaction). Requirements:

- **Exact parity with offline features** (spec 01 AC-2.4). The online store must give the *same* answer as the offline point-in-time computation, so approximate counters are out for the MVP.
- **Idempotent writes.** Kinesis is at-least-once, so a redelivered event must not be counted twice.
- Low latency: a single-digit-ms read within the region, as part of a p95 < 100 ms budget.
- Data older than 24 h is useless and should expire automatically.
- **Budget < $5/month**, ephemeral runtime. Prefer no VPC-bound services, since those would push us towards NAT Gateways or interface endpoints.

## Decision

**Amazon DynamoDB, on-demand capacity, table `velocity`:**

| Attribute | Value |
|---|---|
| PK `card_id` | pseudonymous card token |
| SK `ts_txn` | `"{event_ts_epoch_ms:013d}#{transaction_id}"` (lexicographically sortable) |
| `amount_minor` | int |
| `expires_at` | `event_ts + 26 h` (epoch seconds, **TTL attribute**) |

- **Read:** `Query` with `card_id = :c AND ts_txn BETWEEN :t−24h AND :t` (exclusive upper bound handled in code). The application then **filters by `event_ts` again**, because TTL deletion is asynchronous and expired items can still be returned.
- **Write:** `PutItem` with `ConditionExpression: attribute_not_exists(ts_txn)`. `ConditionalCheckFailedException` = duplicate, which is detected and not double-counted.
- Strongly consistent reads (`ConsistentRead=true`) so the next event on the same card sees the previous write. At demo volume the 2× read cost is negligible.
- The same table (separate item type) or a tiny second table holds the stream consumer **checkpoint**.
- Point-in-time recovery is **off** (ephemeral, rebuildable data). Encryption at rest uses the AWS-owned key.

## Alternatives considered

| Option | Cost (approx.) | Pros | Cons | Why not |
|---|---|---|---|---|
| **DynamoDB on-demand, per-event items + TTL** | ~$0.625/M write units, ~$0.125/M read units (`us-east-1`, after the late-2024 price cut). 25 GB storage free. A demo is cents. **$0 idle.** | Serverless, no VPC, exact windows, conditional writes give idempotency, TTL deletes are free | Query cost grows with events per card per 24 h (hot cards). TTL is lazy. | **Chosen** |
| DynamoDB, minute-bucket counters (`ADD` per bucket) | Same | O(1) item count per window, handles hot cards | Bucket granularity breaks exact parity with offline `[t−w, t)` windows. Idempotency needs an extra dedup record. | Planned for *Later* (spec 01 §11) once parity tolerances are defined. |
| **ElastiCache (Redis/Valkey), sorted sets** | Smallest node `cache.t4g.micro` ~$12/month always-on. Serverless has a minimum billable storage/ECPU (~$6–90/month depending on engine). | Sub-ms latency, natural sliding windows (`ZADD`/`ZRANGEBYSCORE`/`ZREMRANGEBYSCORE`) | **VPC-only.** Fargate in private subnets would need a NAT Gateway (~$32/month) or interface endpoints. Node billing is hourly, not per request. Data is in memory, so durability needs replicas. | Violates the budget and constitution §6. |
| DynamoDB + DAX | DAX node ~$30+/month | µs reads | Cost, VPC-only, and DAX does not help strongly consistent reads | Overkill. |
| In-process state (dict in the consumer) | $0 | Fastest | Lost on restart. Cannot be shared by the API and consumer containers. Cannot scale out. | Not a real design. |
| SageMaker Feature Store (online) | Per-request + storage | Managed offline/online store | Extra service, costs and complexity for 7 features | Revisit for multi-team feature reuse. |

## Consequences

**Positive**
- Exact, testable feature parity: the same `[t−w, t)` semantics as offline.
- Idempotency comes from conditional writes. No separate dedup store.
- Pay-per-request with no idle cost, which suits an ephemeral environment.
- The Fargate task reaches DynamoDB over the public endpoint (TLS + IAM). A free **gateway endpoint** can be added if the task moves to private subnets.

**Negative**
- **Hot cards:** a card with thousands of transactions/24 h makes queries expensive and slow (paginated, 1 MB per page). IEEE-CIS proxy cards are small. Mitigation later: bucketed counters.
- **TTL is lazy.** AWS deletes expired items typically within a few days, not instantly. Correctness therefore depends on the application filter, which a test covers.
- Read-modify-write is not atomic across the read and the put. Concurrent events for the same card on two paths (API + stream) can see a stale history. This is accepted in the MVP (spec 01 §5.1). Partition-key ordering protects the stream path.
- Single-table design is easy to get wrong later. Keep access patterns documented here.

**Revisit if:** p95 read latency exceeds ~20 ms, hot-card queries dominate cost, or the environment becomes always-on with high volume (then compare provisioned capacity or ElastiCache).

## Well-Architected lens

| Pillar | Assessment |
|---|---|
| Operational Excellence | Table, TTL and alarms in CDK. Metrics: `ConsumedRead/WriteCapacityUnits`, `ThrottledRequests`, `ConditionalCheckFailedRequests` (= duplicate rate, a useful business signal). |
| Security | IAM least privilege: `dynamodb:Query`, `PutItem` on this table ARN only. Encryption at rest by default. No PII: `card_id` is a token, amounts only. |
| Reliability | Multi-AZ by default. On-demand absorbs bursts without capacity planning. PITR is off on purpose (data can be rebuilt from the stream/dataset). |
| Performance Efficiency | Key design matches the single access pattern (one card, time range). High-cardinality PK spreads partitions. Strongly consistent reads only where needed. |
| Cost Optimization | On-demand: $0 when idle. TTL deletes cost nothing. Storage stays tiny because of TTL. No NAT/VPC costs. |
| Sustainability | Serverless and shared infrastructure, with no idle nodes. TTL keeps only the data that is needed. |

## SAA-C03 exam notes

- **On-demand vs provisioned:** on-demand for unpredictable or spiky traffic and pay-per-request. Provisioned (+ auto scaling) is cheaper for steady, predictable load. Reserved capacity applies to provisioned only.
- **RCU/WCU:** 1 RCU = 1 strongly consistent read/s of up to 4 KB (or 2 eventually consistent). 1 WCU = 1 write/s of up to 1 KB. Transactional operations cost 2×.
- **TTL:** free deletes, asynchronous (not exact). Expired items appear in DynamoDB Streams as service deletes, which is useful for archiving.
- **DAX** = in-memory cache for DynamoDB (µs, eventually consistent reads). **ElastiCache** = general-purpose Redis/Valkey/Memcached cache. Both run in a VPC.
- **Gateway VPC endpoints** exist only for **S3 and DynamoDB** and are free. Interface endpoints (PrivateLink) cost per hour per AZ plus per GB. This is a classic cost question next to "replace a NAT Gateway".
- **Global Tables** = multi-region active-active. **PITR** = continuous backups (35 days). **GSI vs LSI:** a GSI can be added any time and has its own capacity. An LSI only at creation, with the same PK.
- **Hot partitions:** choose high-cardinality partition keys. Use write sharding (suffixes) when one key is hot.
