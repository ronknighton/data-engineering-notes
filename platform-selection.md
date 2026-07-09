# Platform Selection: Snowflake vs Databricks vs AWS-native

## Core framing
Most DE roles inherit an existing stack rather than choosing one. This
framework's real value is being able to explain *why* an existing choice
fits a workload — not picking greenfield.

Databricks/Snowflake are governed platforms that bundle infra assembly
(secrets, IAM, networking, catalog) you'd otherwise wire yourself in AWS.
That bundling is a real convenience — paid for with a platform premium
and, in Databricks' case, some lock-in on the governance layer even when
the underlying files (Delta on S3) stay portable.

## Snowflake wins when:
- Team is SQL-first (analysts, analytics engineers) — no Python/Spark needed
- Workload is structured, tabular, query-heavy (joins/aggregations feeding BI)
- Cost predictability matters more than minimization (per-second billing,
  auto-suspend — easy to cap vs. Databricks cluster misconfig risk)
- Governance is warehouse-native, no need to unify across external platforms
- CDC/streaming into curated structured tables at moderate volume
  (Snowpipe Streaming / dynamic tables have closed most of the old gap)
- No dedicated platform-ops role — Snowflake needs far less operational tuning

## Databricks wins when:
- Transformation logic can't be expressed cleanly in SQL (fuzzy matching,
  custom dedup heuristics, iterative/graph algorithms) — Snowpark narrows
  but doesn't eliminate this gap
- Data is semi-structured/unstructured at real volume (JSON, logs, ML inputs)
- ML/data science shares the same data — MLflow, feature store, same
  Unity Catalog tables avoid re-exporting curated data into a separate env
- Need ACID MERGE on files you own in open format (Delta on your own S3) —
  portable if another engine needs to read the same files directly
- True massive scale with complex multi-stage transforms — more direct
  control over partitioning/shuffle than Snowflake's optimizer offers
- Streaming needs complex stateful logic (windowing, sessionization,
  multi-stream joins) — not just "keep this table fresh"
- Multi-cloud/cloud-agnostic requirement

## AWS-native (Lambda/Glue/Step Functions) wins when:
- Volume is low/bursty — no cluster spin-up floor or warehouse resume
  minimum to pay for
- Transform is simple (extract → light reshape → load) — nothing benefits
  from a distributed engine or SQL optimizer
- Append-only/full-refresh is fine — no need for merge/ACID semantics
  (Glue now supports Iceberg tables, narrowing this further)
- Team already fluent in IAM/networking — not buying meaningful time savings
  from the platform premium
- Simple orchestration (few sequential/parallel steps) — Step Functions
  beats standing up Airflow/Databricks Workflows for that scale
- Cost line-items need to map cleanly to existing AWS spend commitments
  (RIs, Savings Plans, EDP) rather than a separate DBU/credit contract

## The tiebreaker
Does the workload need ACID/merge semantics, distributed compute at scale,
or shared ML tooling badly enough to justify the platform premium over
hand-building that plumbing yourself? If yes → Databricks or Snowflake
depending on SQL vs. custom-code shape. If no → AWS-native is usually
cheaper, and the "harder" IAM/VPC setup is often deliberate friction in a
regulated shop, not a deficiency.

## Common real-world pattern
Many orgs run **both**: Databricks for ingestion/bronze/silver + ML,
Snowflake (or Databricks SQL warehouses) for the BI-facing gold layer.
"Best tool per layer" is a stronger interview answer than picking one
platform for an entire pipeline, since the two platforms' strengths are
largely complementary rather than overlapping.

## Interview note
When asked "why Databricks/Snowflake here?" — the answer isn't "it's the
industry standard," it's tracing the specific workload need (SQL-shaped
vs. custom-code-shaped, structured vs. semi-structured, ACID-need
vs. append-only) to the platform that matches it.