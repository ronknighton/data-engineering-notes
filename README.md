# Data Engineering Notes

Distilled reference notes from DEA coursework and working sessions. Each entry follows the same shape: **the concept → the why (what the curriculum didn't explain) → production framing → interview one-liner.**

This is the reference layer, not the archive. Chats and videos are the archive; these notes are what gets reread.

## Index

| File | Covers |
|---|---|
| [snowflake.md](snowflake.md) | RBAC / grants, Partner Connect isolation, JS stored procedures, streams & tasks, Time Travel, COPY INTO metadata |
| [dbt.md](dbt.md) | Layer ownership, sources vs. models, `generate_schema_name`, pre-hooks, snapshots (SCD2), vars, query tags, Fusion 2.0 changes, Connection Profiles |
| [aws.md](aws.md) | IAM authN vs. authZ, S3 policy scoping, S3 as decoupling layer, Glue vs. Lambda for extraction, Kinesis Streams vs. Firehose, DMS/CDC |
| [spark-databricks.md](spark-databricks.md) | UDF types, data skew & salting, Delta Lake, medallion architecture, Unity Catalog gap |
| [interview-narratives.md](interview-narratives.md) | Reusable framings: staged modernization, cost-as-constraint, vendor bias, logical vs. physical modeling, narrating decisions |

## Conventions

- One file per major topic; one `##` heading per concept.
- Backfill opportunistically — when a topic resurfaces, that's the trigger to write or update its entry. No big-bang migrations.
- End-of-session habit: ask Claude to distill 3–5 takeaways formatted for this repo, paste, commit.
