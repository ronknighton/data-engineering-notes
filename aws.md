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
