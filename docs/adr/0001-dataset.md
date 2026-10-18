# 0001 — Training dataset: IEEE-CIS Fraud Detection

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-10-18 |
| **Deciders** | Kacper Celejewski |
| **Related** | [spec 01 §4](../../specs/01-mvp.md) |

## Context

The project needs a public, labelled payment-fraud dataset that supports:

1. **Time-based (out-of-time) splits** and **simulated label delay**, so it needs a usable time column covering several months.
2. **Velocity features per card/account**, so it needs some kind of entity identifier.
3. **Credible results.** A dataset where any model reaches 0.99+ AUC teaches nothing and looks naive to a reviewer from the fraud domain.
4. A **card-payment** context, matching the target role (ML in payments).

The two most common candidates are PaySim and IEEE-CIS. (The ULB "Credit Card Fraud" dataset, about 284k rows, was set aside early: it covers only 2 days and its features are already PCA-transformed, so neither time splits nor velocity features are possible.)

## Decision

Use **IEEE-CIS Fraud Detection** (Vesta Corporation, Kaggle 2019), `train_transaction.csv` + `train_identity.csv` only.

- Time: `TransactionDT` (a seconds offset, about 6 months) is mapped onto a synthetic UTC calendar.
- Entity: a pseudonymous `card_id` built from `card1`, `addr1` and the derived first-seen day `floor(TransactionDT/86400) − D1`.
- The raw data is **not committed** (Kaggle competition rules). The repo has download instructions (Kaggle CLI) and SHA-256 checksums.
- PaySim *may* be used later as a synthetic load generator, but never for reported metrics.

## Alternatives considered

| | **IEEE-CIS (Vesta)** | **PaySim** (synthetic mobile money) |
|---|---|---|
| Origin | Real e-commerce card-not-present transactions, anonymised | Agent-based simulation calibrated on one African mobile-money operator |
| Size / period | ~590k labelled rows, ~6 months | ~6.3M rows, 744 hourly steps (~30 days) |
| Fraud rate | ~3.5% | ~0.13% |
| Time column | `TransactionDT`, second resolution | `step`, hour resolution |
| Entity ID | None explicit; `card1..6`, `addr1`, `D1` let you build a proxy | `nameOrig` / `nameDest` |
| Feature realism | High, but ~400 columns anonymised (`C*`, `D*`, `M*`, `V*`) | Low: 5 transaction types, fraud only in `TRANSFER`/`CASH_OUT` |
| Leakage traps | Kaggle-style aggregations over train+test; label propagation per account | **Balance columns** (`oldbalanceOrg`, `newbalanceOrig`) almost reveal the label: fraud empties the account |
| Typical result | PR-AUC well below 1, so there is room to show real trade-offs | ~0.99+ AUC with trivial models |
| Domain fit | Card payments (target domain) | Mobile money / wallets |
| Licence | Kaggle competition rules (download per user, no redistribution) | CC BY-SA 4.0 |

**Why not PaySim:** the fraud pattern is too easy to learn, the balance fields leak the label, it covers only about 30 days (so no realistic 30–90 day label delay is possible), and it is not card payments. Results on PaySim would undermine the project's main claim, which is honest evaluation.

## Consequences

**Positive**
- Results are believable, and the gap between naive and leakage-free evaluation can actually be seen (experiment E-LBL).
- 6 months is enough for train / valid / 30-day label gap / test (spec 01 §4.3).
- Card-not-present e-commerce is close to real PSP/acquirer fraud work.

**Negative / risks**
- **No true card ID.** The `card_id` proxy will sometimes merge or split real cards, so velocity features are noisier than in production. This is documented as a limitation.
- **Anonymised features** make reason codes less meaningful (`OTHER_RISK_SIGNALS` bucket, spec 01 §4.7).
- **Label semantics:** Vesta's labelling reportedly marks later transactions of an account as fraud after a chargeback, so labels are partly account-level. A time split reduces but does not remove the overlap of entities between TRAIN and TEST. This is reported as a known risk.
- Amounts are in USD. Costs are converted to PLN at a configurable fixed rate.
- **Licence:** users must accept the competition rules on Kaggle themselves. CI cannot download the data, so CI tests use small synthetic fixtures.

**Revisit if:** a public card dataset with real entity IDs and real label timestamps becomes available.

## Well-Architected lens

The dataset choice is mostly an ML decision. The [Machine Learning Lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/machine-learning-lens.html) of Well-Architected applies here.

| Pillar | Assessment |
|---|---|
| Operational Excellence | Reproducible data prep from the raw files with checksums. Every MLflow run records the dataset version. |
| Security | No real cardholder data. Raw data stays local (or in a private S3 bucket with Block Public Access if it is ever uploaded). Pseudonymous `card_id` only. |
| Reliability | Deterministic, seeded prep and label-delay simulation, so a failed run can be repeated exactly. |
| Performance Efficiency | ~590k rows fit in laptop memory (pandas/polars). No distributed processing needed. |
| Cost Optimization | Training runs locally, so the AWS cost of the ML phase is $0. |
| Sustainability | A small dataset and a small model mean minimal compute. Prepared data is cached as Parquet, not recomputed. |

## SAA-C03 exam notes

- The exam does not test datasets, but it tests **data protection**: S3 Block Public Access, SSE-S3 vs SSE-KMS, bucket policies, and Macie for discovering PII in S3.
- Know where ML services fit: **SageMaker** (build/train/deploy), **Amazon Fraud Detector** (managed fraud service; AWS has announced it is closed to new customers, so check the current status. It is a good example of "managed vs build your own"), **Glue** (ETL/catalog), **Athena** (SQL over S3), which is how the decision log would be queried in a production setup.
