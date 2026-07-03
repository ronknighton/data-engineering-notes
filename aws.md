# AWS

## IAM: authentication vs. authorization

**Concept.** `aws configure` stores an access key pair → boto3 authenticates → AWS knows *who* you are. Whether you can *do* anything is a separate check against identity-based policies attached to the user/group/role. `AccessDenied` means authN succeeded, authZ failed (`InvalidAccessKeyId` / `SignatureDoesNotMatch` = authN failed).

**Real incident.** `S3UploadFailedError ... not authorized to perform: s3:PutObject ... because no identity-based policy allows` — fixed by attaching a scoped policy to the IAM user. No credential changes needed; policy changes take effect in seconds.

**Least-privilege pattern.** Scope to the bucket, and note the Resource split: `s3:ListBucket` applies to the bucket ARN, `s3:PutObject`/`GetObject` apply to `arn:...:bucket/*`. Mixing those up causes confusing partial-access errors.

**Interview one-liner.** "AccessDenied is an authorization failure, not an authentication one — the fix is a policy grant, not new credentials, and least privilege means scoping actions and resources, not attaching S3FullAccess."

**Bonus trace-reading detail.** boto3 auto-switches to multipart upload above ~8 MB — a failure on `CreateMultipartUpload` vs. `PutObject` hints at file size, same root cause.

## S3 as the universal decoupling / replay layer

Landing everything in S3 between pipeline stages decouples producers from consumers, enables replay/backfill, and is dirt cheap relative to compute. Cost of the pattern: data multiplies across stages (raw copy, transformed copy, warehouse copy) — storage multiplication is a real line item to acknowledge.

## Glue vs. Lambda vs. managed connectors for extraction

For API → S3 extraction: Glue **Python Shell** jobs (not Spark jobs) fit small/medium pulls; Lambda fits if payload and runtime fit its limits (15-min max); Fivetran/managed connectors trade money for zero ops. Choosing Spark-sized tools for API pulls is over-provisioning — the workload defines the tool.

## Kinesis Data Streams vs. Firehose

Distinct services: Streams = real-time, consumer-managed, shard-based, **not free-tier eligible** (watch spend on DEA projects); Firehose = near-real-time managed delivery to S3/Redshift/etc. Firehose was renamed **Amazon Data Firehose** (late 2023) — use current name in interviews.

## DMS / CDC

DMS replicates changes with an Op flag (I/U/D) in its S3 output files — the raw material downstream SCD logic consumes.

## S3 CRR vs. event-driven cross-region copy

**Concept.** S3 Cross-Region Replication (CRR) is native, automatic, zero-compute
replication: enable versioning, define a replication rule, done. An SNS → SQS →
Lambda pipeline doing `copy_object` re-implements this with owned code.

**The why (curriculum gap).** DEA's DMS project uses the event-driven pipeline for
pure copy — no transformation. That's a teaching choice: the SNS/SQS/Lambda fan-out
pattern is the canonical AWS event architecture (SNS = fan-out to many subscribers;
SQS = durability, retry, backpressure). The stated problem is a CRR problem.

**Production framing.** The pipeline earns its complexity only with transformation,
routing, fan-out, or downstream triggering in the replication path. Caveats worth
knowing: CRR requires versioning, only replicates objects created after the rule
exists (backfill = S3 Batch Replication), and its retry internals are opaque —
the custom pipeline's DLQ/metrics visibility is its one honest advantage. Cost is
mostly a wash (inter-region transfer dominates either way); the real difference is
operational surface.

**Interview one-liner.** "If bytes just need to exist in two regions, CRR — zero
code to operate. The event pipeline is justified only when the copy path needs
compute."

## CDC to S3: real-time capture ≠ real-time files

**Concept.** CDC (e.g., DMS reading the transaction log) captures every change in
real time — near-zero source load, sees deletes (Op flag I/U/D), preserves
intermediate states that periodic extracts miss. But flushing one file per change
creates the small-file problem: task overhead in Spark, per-file costs in
Athena/Glue listing and Snowpipe ingestion.

**The why.** "Real-time" describes the capture layer, not the write cadence. DMS's
S3 target has micro-batching settings (`cdcMaxBatchInterval`, `cdcMinFileSize`)
that buffer changes and flush on time-or-size thresholds — same buffer-then-flush
idea as Firehose. Residual small files get fixed by periodic compaction (why Delta
Lake has `OPTIMIZE`).

**Interview one-liner.** "You almost never want real-time files — buffer at the
delivery layer, trading seconds of latency for sane file sizes, because small
files tax every downstream consumer."
