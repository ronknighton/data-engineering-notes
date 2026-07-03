# Interview Narratives

Reusable framings — my own positions, developed through DEA coursework, not recitations.

## Staged modernization framework (mine)

Evaluate migrations in three tiers rather than as a binary rewrite: **budget tier** (squeeze existing stack: spot instances, right-sizing, incremental fixes), **hybrid tier** (replace the bottleneck component, keep the rest — e.g., Snowflake + dbt on top of existing ingestion), **full refactor** (platform migration, e.g., Databricks lakehouse) — justified only when the middle tier can't meet requirements. Presents as: "before recommending the target architecture, I'd establish which tier the constraints actually demand."

## Cost is a business constraint, not a secondary concern

Pushback on "business problem first, cost is not a problem": cost **is** a business problem. The strongest framing treats business requirements and cost as simultaneous constraints, not a priority ordering. In design interviews, state assumed budget posture explicitly as one of the requirements.

## Vendor bias drives architecture more than technical merit

Across multiple case studies (Azure Databricks retail migration, Postgres→Snowflake, Fortune 500 health services, clickstream analytics): existing vendor relationships and ecosystem lock-in explain technology choices at least as well as technical fit. Interview use: acknowledge it without dismissiveness — "the choice of X here likely reflects an existing Azure relationship; on technical merit alone, Y would also be defensible."

## Logical vs. physical modeling vocabulary

Say "entity" during conceptual/logical discussion, "table" during physical implementation. The vocabulary shift itself signals modeling maturity.

## Narrate decisions and assumptions aloud

Flagging assumptions and reasoning in real time matters as much as the destination. Applies doubly to trade-offs: name what's being given up, not just what's chosen.

## Pattern recognition > tool recitation

Recurring meta-skill: distinguish which parts of a working system are principled and which are workshop conveniences (e.g., dbt pre-hook ingestion works, but production wants Snowpipe/orchestration). Being able to say "this works here, and here's what I'd change in production and why" is the differentiator.

## Concrete stories to have ready

- **S3 AccessDenied debugging:** read the trace, distinguished authN from authZ, wrote a least-privilege bucket policy instead of attaching S3FullAccess.
- **dbt Fusion breaking change:** diagnosed dbt1159 (`accepted_values` args nesting), understood it as config-vs-arguments separation, fixed and validated across 7 tests.
- **Partner Connect grants:** traced "mystery" permission steps to role isolation; articulated the grant hierarchy.
- **MERGE + stream filter bug:** moved `METADATA$ACTION='INSERT'` into the USING subquery — example of hypothesis-driven debugging.
- **Platform drift vs. stale training material:** repeatedly reconciled outdated videos with current UIs (Connection Profiles, RSA key pair auth) — evidence of self-sufficiency with evolving tools.
