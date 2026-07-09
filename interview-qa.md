# Data Engineering Interview Q&A

Source of truth for interview prep. Paste-and-commit like the rest of the
knowledge repo — edit freely, this isn't regenerated wholesale.

Sections 1–4 are pulled from actual project work (higher value — these are
the differentiated, "why" answers). Section 5 is data modeling/system design
fundamentals. Section 6 is condensed general-knowledge Q&A from GeeksforGeeks,
kept for breadth/warm-up recall.

---

## 1. Snowflake

**Q: Are Snowflake stored procedures compiled at call time, or is there a persistent object?**
A: Persistent object, same as SQL Server. `CREATE OR REPLACE PROCEDURE` stores
the definition permanently in the schema (`SHOW PROCEDURES IN SCHEMA ...` or
`INFORMATION_SCHEMA.PROCEDURES` to verify). The JavaScript body is stored as
text in metadata; Snowflake compiles and executes it at call time. Key
difference from T-SQL: procedures are overloaded by argument signature, so
`CUSTOMER_SP()` and `CUSTOMER_SP(VARCHAR)` are distinct objects.

**Q: How do you flatten a nested JSON VARIANT column in Snowflake?**
A: `LATERAL FLATTEN(input => column)`. For deeply nested JSON (arrays inside
arrays), chain multiple `LATERAL FLATTEN` calls — one per nesting level.
The `.value` accessor pulls the actual value out of the metadata columns
FLATTEN produces (`SEQ`, `KEY`, `PATH`, `INDEX`, `VALUE`, `THIS`).

**Q: RANK() vs DENSE_RANK() vs ROW_NUMBER() — when do you use each?**
A: `ROW_NUMBER()` always gives unique sequential integers, even for ties —
use when you need exactly one row per group (e.g. dedup). `RANK()` gives
ties the same rank but skips subsequent numbers (1,1,3). `DENSE_RANK()`
gives ties the same rank with no gaps (1,1,2). Use `DENSE_RANK()` for "top N
distinct values" style questions; `RANK()` when the skipped numbers should
reflect competition-style standing.

**Q: How do you query point-in-time data in Snowflake?**
A: Time Travel — `SELECT ... FROM table BEFORE (STATEMENT => 'query_id')`,
or `AT`/`BEFORE (TIMESTAMP => ...)`. Useful for recovering dropped/modified
rows without a separate backup process; can reconstruct missing records via
`LEFT JOIN` against the historical snapshot.

**Q: How do Streams and Tasks implement change-driven processing (e.g. SCD1 via MERGE)?**
A: A Stream captures row-level changes (`METADATA$ACTION`, `METADATA$ISUPDATE`)
on a source table since it was last consumed. A Task runs on a schedule and
executes a `MERGE` against the stream. Gotcha: filter `METADATA$ACTION =
'INSERT'` inside the `USING` subquery, not in the `WHEN MATCHED`/`WHEN NOT
MATCHED` clauses — filtering in the wrong place silently drops new records
from being picked up.

**Q: What changed with Snowflake credential auth in dbt platform?**
A: New Snowflake credentials in dbt platform now default to RSA key-pair
auth — username/password is no longer accepted for new credentials. Requires
generating an RSA key pair and running `ALTER USER ... SET
RSA_PUBLIC_KEY=...` in Snowflake before the credential will save.

**Q: What's the difference between star schema and snowflake schema?**
A: Star schema: one central fact table, denormalized dimension tables
directly around it — fewer joins, faster reads, some redundancy. Snowflake
schema: dimension tables are further normalized into sub-dimensions —
less redundancy, more joins. Interview framing: star schema optimizes for
query simplicity/speed (typical in Snowflake/BI-facing gold layers);
snowflake schema optimizes for storage/integrity at the cost of join
complexity.

---

## 2. dbt

**Q: How does dbt implement SCD Type 2, and what's `check` vs `timestamp` strategy?**
A: The `{% snapshot %}` block. With `strategy='check'` and a `check_cols`
list, each `dbt snapshot` run compares current rows against the snapshot
table by primary key; if any checked column changed, it closes the old row
(`DBT_VALID_TO`) and inserts a new one (`DBT_VALID_FROM`). `timestamp`
strategy instead relies on an `updated_at` column to detect change rather
than comparing column values directly — cheaper to evaluate but only works
if the source reliably maintains that column. Gold-layer views typically
rename `DBT_VALID_FROM/TO` to business-friendly version-date columns and
coalesce open-ended `NULL` valid-to into a sentinel like `9999-12-31`.

**Q: What does `generate_schema_name` do and why override it?**
A: It's a dbt macro controlling how the physical schema name is built from
the `dbt_project.yml`/model config. Overriding it is common when you want
env-specific or layer-specific schema naming instead of dbt's default
`{target_schema}_{custom_schema}` concatenation behavior.

**Q: What are `vars` in dbt used for, vs parameters?**
A: `vars:` in `dbt_project.yml` are project-level configuration values
(stage name, database, file format) referenced via `{{ var('...') }}` in
models/macros — lets you repoint a macro at different environments by
changing one block rather than hardcoding. Different from Jinja macro
parameters, which are scoped to a single macro call.

**Q: dbt doesn't load data — so what's a pre-hook loading pattern, and why is it a trade-off?**
A: A pre-hook runs arbitrary SQL before a model materializes. Some projects
wrap a `DELETE` + `COPY INTO` in a macro and invoke it as a model's
pre-hook, effectively using dbt as a lightweight loader/orchestrator. Works
for small/self-contained pipelines, but blurs the layer-ownership
principle — in production, ingestion would typically be Snowpipe, an
Airflow task, or a dedicated ingestion tool, with dbt starting at the
transform layer. It hides ingestion inside a transform tool where it's
harder to monitor, retry, or schedule independently.

**Q: CTE vs plain SELECT in a dbt model — functional difference?**
A: None for simple models — CTEs and plain selects compile to equivalent
SQL. The value of CTEs is readability and debuggability (you can run each
CTE block standalone to isolate a bug), not performance.

**Q: `dbt run` vs `dbt build`?**
A: `dbt run` only materializes models. `dbt build` runs models, tests,
snapshots, and seeds together in DAG order — fails fast if a test on an
upstream model fails, preventing bad data from propagating downstream in
the same invocation.

**Q: Common `schema.yml` source-naming mistake?**
A: The `name:` field under `sources:` must match the *source* name (the
first argument to `{{ source('name', 'table') }}`), not the model name
that consumes it. Using the model name there breaks the reference silently
until you try to compile.

---

## 3. AWS

**Q: 401 vs 403 — what's the distinction and where does it show up?**
A: 401 Unauthorized = identity not established (missing/invalid/expired
credentials). 403 Forbidden = identity established, action not permitted
(valid credentials, insufficient policy). Same two-layer model shows up
across AWS IAM (`AccessDenied` on a valid identity), Snowflake, GitHub, and
most modern APIs — e.g. a Calendly `insufficient_scope` error is a
403-class failure: the token authenticated fine, the scope just didn't
cover the endpoint.

**Q: Why doesn't Lambda pick up a new image after `docker push` to ECR?**
A: Lambda doesn't auto-detect new ECR pushes. You must explicitly run
`aws lambda update-function-code --function-name ... --image-uri ...`
after every push. Trips people up the first time because zip-based Lambda
deploys feel more "automatic" by comparison.

**Q: Docker-based Lambda vs zip deployment — what's the trade-off?**
A: Docker trades slightly slower cold starts (larger image = longer pull
at cold start) for full environment control, reproducible builds, and a
10GB size ceiling (vs zip's much smaller limit). Zip deployment stays
reasonable for lightweight functions with minimal dependencies.

**Q: Kinesis Data Streams vs Kinesis Firehose — same thing?**
A: No — distinct services. Data Streams is a low-latency, custom-consumer
streaming service you write processing logic against (Lambda, KCL app,
etc.) — you manage shards and scaling. Firehose is a fully managed
delivery service that buffers and batches records into a destination (S3,
Redshift, OpenSearch) with built-in transformation — no consumer code
needed, less control over latency/processing.

**Q: How does DMS represent change events in CDC output, and what's the "small-file problem"?**
A: DMS CDC output files carry an `Op` flag per record — `I` (insert), `U`
(update), `D` (delete) — which downstream processing keys off to apply the
right operation. The small-file problem arises because CDC capture cadence
(near-continuous) often outpaces a sensible write cadence to S3 — writing
every change event as its own tiny file creates massive object-count
overhead and hurts downstream query performance. Mitigation: micro-batching
writes on a time or size threshold rather than per-event.

**Q: S3 Cross-Region Replication (CRR) vs an event-driven copy pattern — when would you pick each?**
A: CRR is passive, automatic, near-real-time bucket-to-bucket replication —
simplest option when you just need a copy in another region with no
transformation. Event-driven (S3 event → Lambda/SNS/SQS → copy) buys you
control: you can filter, transform, fan out to multiple destinations, or
add retry/DLQ logic. Pick CRR for straightforward DR/compliance copying;
pick event-driven when the copy needs conditional logic or isn't a 1:1
mirror.

**Q: How do you diagnose an S3 `AccessDenied` on `CreateMultipartUpload` with valid credentials?**
A: Valid credentials ≠ authorized action — this is IAM authorization, not
authentication. Check whether the IAM identity has an attached
identity-based policy (or the bucket has a resource-based policy) granting
`s3:PutObject` (and related multipart actions) on that specific
resource/prefix. A user can authenticate fine and still get denied if no
policy explicitly allows the action.

**Q: Why is AWS Glue scoped narrowly to just the extraction step in a mart-layer pipeline (API → S3 → Snowflake → dbt)?**
A: Glue's Spark engine is overkill for a simple API pull — a Python Shell
job (or even Lambda, depending on payload size/execution time) handles
extraction more cheaply. Once data lands in S3 and is loaded into
Snowflake, dbt owns raw → transform → mart, which keeps each tool doing
what it's actually good at rather than making Glue do transformation work
a SQL-native tool handles better.

**Q: What's the staged-modernization framing for evaluating a legacy-to-cloud migration?**
A: Budget tier → hybrid tier → full refactor. Budget tier: minimal-cost,
often lift-and-shift or thin wrapper changes. Hybrid tier: selectively
modernize the highest-value/highest-pain components while leaving stable
legacy pieces alone. Full refactor: complete re-architecture, highest cost
and risk, justified only when the legacy system is a genuine blocker. This
framing signals cost-consciousness as a first-class constraint rather than
defaulting to "rip and replace."

---

## 4. Platform Selection: Snowflake vs Databricks vs AWS-native

**Q: When does Snowflake clearly win over Databricks?**
A: Team is SQL-first (analysts, not Python/Spark engineers); workload is
structured/tabular/query-heavy; cost predictability matters more than
minimization (per-second billing, auto-suspend); governance is
warehouse-native with nothing external to unify; CDC/streaming into
structured tables at moderate volume (Snowpipe Streaming/dynamic tables);
no dedicated platform-ops role to tune clusters.

**Q: When does Databricks clearly win over Snowflake?**
A: Transformation logic can't be expressed cleanly in SQL (fuzzy matching,
custom dedup, iterative/graph algorithms); data is semi/unstructured at
real volume; ML/data science shares the same data (MLflow, feature store,
same Unity Catalog tables); need ACID MERGE on files you own in an open
format (Delta on your own S3); true massive scale with complex multi-stage
transforms; streaming needs complex stateful logic (windowing,
sessionization); multi-cloud requirement.

**Q: When is plain AWS-native (Lambda/Glue/Step Functions) cheaper than either platform?**
A: Low/bursty volume (no cluster spin-up or warehouse-resume floor to pay
for); simple single-purpose transforms with no need for a query
optimizer/distributed engine; append-only or full-refresh is acceptable
(no merge/ACID need); team already fluent in IAM/networking so the
platform isn't buying meaningful time savings; simple orchestration (Step
Functions vs. standing up Airflow/Workflows); cost needs to map cleanly to
existing AWS spend commitments (RIs/Savings Plans) rather than a separate
DBU/credit contract.

**Q: What's the actual tiebreaker for platform choice?**
A: Does the workload need ACID/merge semantics, distributed compute at
scale, or shared ML tooling badly enough to justify the platform premium
over hand-building that plumbing yourself? If yes → Databricks or
Snowflake depending on whether the work is SQL-shaped or custom-code-shaped.
If no → AWS-native is usually cheaper, and the "extra" IAM/VPC setup work
is often deliberate governance friction in a regulated shop, not a
deficiency.

**Q: What's a common real-world pattern that avoids picking just one platform?**
A: Many orgs run both — Databricks for ingestion/bronze/silver + ML,
Snowflake (or Databricks SQL warehouses) for the BI-facing gold layer.
"Best tool per layer" is a stronger interview answer than defending one
platform for an entire pipeline, since the two platforms' strengths are
largely complementary rather than overlapping.

**Q: Databricks Unity Catalog "external connection" ease vs AWS — is Databricks actually ahead here?**
A: Not a capability gap — a surface-area difference. A UC Connection
bundles credential storage + RBAC + network egress into one securable
catalog object. AWS has the same pieces (Glue Data Catalog Connections +
Secrets Manager + IAM, or EventBridge API Destinations) but spread across
services you assemble yourself. UC's win is governance unification (one
`GRANT`, one audit log) at the cost of platform lock-in — those
connection definitions don't travel if you leave Databricks.

---

## 5. Data Modeling & System Design Fundamentals

**Q: Entity vs table — why does the distinction matter in an interview?**
A: "Entity" is the conceptual/logical modeling term; "table" is the
physical implementation. Using them interchangeably reads as less mature —
using "entity" when discussing a data model and "table" when discussing
its physical implementation signals you understand the modeling layers are
distinct steps, not the same thing.

**Q: What are the three SCD types?**
A: Type 1: overwrite the old value, no history. Type 2: insert a new row
per change, preserving full history (what dbt snapshots implement). Type
3: add a new column to track a limited history (e.g. "previous value"
column) — rarely used beyond simple single-hop change tracking.

**Q: Data lake vs data warehouse?**
A: Warehouse: structured data, schema-on-write, optimized for analysis,
typically consumed by business analysts. Lake: structured + semi- +
unstructured data, schema-on-read, serves as a repository for raw data,
typically consumed by data scientists/engineers. Lakehouse architectures
(Databricks/Delta) attempt to merge both — warehouse-like guarantees on
lake-native open file formats.

**Q: Normalization vs denormalization trade-off?**
A: Normalization reduces redundancy and improves integrity by splitting
data into focused related tables — costs you joins. Denormalization
improves query performance and simplifies queries by reducing joins —
costs you redundancy, more complex updates, and risk of inconsistency.
Gold/BI-facing layers commonly denormalize deliberately; raw/silver layers
usually stay normalized.

**Q: What is the Lambda architecture (the pattern, not AWS Lambda)?**
A: A data processing pattern combining batch and stream processing: a
batch layer manages the master dataset and pre-computes batch views, a
speed layer handles real-time processing for freshness, and a serving
layer merges results from both to answer queries. Largely superseded in
modern practice by Kappa-style single-pipeline streaming architectures,
but still a common interview reference point.

**Q: Common data partitioning strategies and why partition at all?**
A: Range, hash, and list partitioning. Partitioning improves query
performance (prune irrelevant partitions), enables parallel processing,
and keeps very large tables manageable. Choice depends on query pattern —
range partitioning suits time-series/date-filtered queries; hash
partitioning suits even distribution when there's no natural range key.

**Q: How do you handle data skew in distributed processing?**
A: Identify the skewed key(s); apply salting/hashing to spread a hot key
across more partitions; use broadcast joins when one side of a join is
small; adjust partition sizing or use a custom partitioner; consider
two-phase aggregation for skewed aggregations rather than a single-pass
groupby.

**Q: How do you handle schema evolution in a pipeline?**
A: Prefer schema-on-read formats (Parquet, Avro, Delta) that tolerate
additive changes; design for backward/forward compatibility; use a schema
registry for centralized versioning where multiple consumers depend on a
shared schema; test schema changes against downstream consumers before
deploying; have a migration plan for breaking (non-additive) changes.

---

## 6. General Data Engineering Knowledge (condensed reference)

**Q: What is data engineering, and how does it differ from data science?**
A: Data engineering builds and maintains the systems that collect, store,
and move data reliably — pipelines, warehouses, quality, accessibility.
Data science analyzes that data to build models and extract insights. DE
is the infrastructure the DS role depends on.

**Q: What is an ETL pipeline, and how does ELT differ?**
A: ETL: extract from source, transform before loading into the target.
ELT: load raw data first, transform inside the destination (common with
modern cloud warehouses like Snowflake, where compute is cheap and
elastic) — this is effectively what a dbt-centric pipeline does: land raw
data, transform in-warehouse.

**Q: SQL vs NoSQL — core differences?**
A: Structure (fixed schema vs schema-less/flexible), scalability
(vertical-leaning vs horizontally scalable), data model (tables/rows vs
document/key-value/graph/column-family), and ACID guarantees (typically
strong in SQL, often traded for availability/partition tolerance in
NoSQL per CAP theorem).

**Q: What is data governance, in one line?**
A: The processes, roles, policies, and standards that ensure data is used
effectively, securely, and compliantly across an organization.

**Q: What is data lineage and why does it matter?**
A: The tracked lifecycle of data — origin, movement, transformations,
downstream impact. Matters for impact analysis before making changes,
regulatory compliance/auditing, and debugging data quality issues back to
their source.

**Q: Batch vs stream processing?**
A: Batch: high-volume jobs processed on a schedule/trigger, efficient when
immediate results aren't required. Stream: continuous processing as data
arrives, enabling real-time or near-real-time action.

**Q: What's the role of a schema registry in streaming pipelines?**
A: Centralizes schema definitions for producers/consumers on a stream
(e.g. Kafka), enforcing compatibility rules so a producer can't push a
breaking schema change without consumers being able to detect and handle
it.

**Q: Common data quality assurance approaches in ETL?**
A: Validation at both source and target, data profiling to understand
characteristics before building rules, cleansing/standardization steps,
reconciliation checks between source and target counts, and a defined
process for triaging and resolving quality issues rather than ad hoc
fixes.

**Q: Behavioral — how do you approach learning new tools in a fast-moving field?**
A: Frame around a concrete recent example: identify the tool via a real
project need, build a minimal proof-of-concept before committing time,
lean on official docs over blog posts for anything going into production,
and validate understanding by explaining the trade-off out loud (interview
tip: this is literally what the "push past the DEA's what, ask about the
why" habit demonstrates — use a real instance of that as the story).

**Q: Behavioral — how do you prioritize tasks on a data engineering project?**
A: Business impact and urgency first, then dependencies between tasks
(what's blocking what), then resource/time constraints. Cite a specific
framework only if you actually use one (Eisenhower Matrix, MoSCoW) —
otherwise describe the reasoning in plain terms, since naming a framework
without applying it convincingly reads as recitation.

---

*Next additions: Airflow and remaining dbt modules once completed. Add
Databricks/Delta Lake/Unity Catalog depth once covered — flagged gap for
non-AWS-native company interviews.*
