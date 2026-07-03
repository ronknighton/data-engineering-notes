# Snowflake

## RBAC: authentication vs. authorization

**Concept.** Snowflake access control is role-based and default-deny. A role sees nothing it hasn't been granted, and grants are hierarchical: USAGE on the database → USAGE on the schema → privileges on the object (SELECT/INSERT/etc.), plus USAGE on stages and file formats. Missing any level of the chain breaks access.

**The why (curriculum gap).** DEA docs say "click Grant Privileges" without explaining *why* grants are needed at all. The reason is always the same: the connecting identity's role doesn't own or have visibility into the target objects.

**Production framing.** ACCOUNTADMIN-for-everything (my training setup) is an anti-pattern; real shops create purpose-scoped roles (LOADER, TRANSFORMER, REPORTER) and grant along the hierarchy. Also: `CREATE OR REPLACE` drops and recreates an object, which can wipe existing grants — a classic "worked yesterday" trap. Order of operations matters.

**Interview one-liner.** "Snowflake grants cascade down a hierarchy — database USAGE, schema USAGE, then object privileges — and replacing an object can silently drop its grants."

## Partner Connect creates an isolated identity

**Concept.** Provisioning dbt via Partner Connect auto-creates a bundle: `PC_DBT_DB`, `PC_DBT_WH`, a service user, and `PC_DBT_ROLE`. dbt Cloud connects **as that role**, not as me.

**The why.** This is the entire reason the SCD2 project needs "extra" permission steps: my bronze objects live in `MYDB.BRONZE` under a different role, so `PC_DBT_ROLE` needs explicit grants (database USAGE, schema privileges, stage USAGE) before dbt can touch them.

**Interview one-liner.** "Partner Connect provisions a separate role, so cross-database access requires deliberate grants — same authN-vs-authZ distinction as an IAM AccessDenied in AWS."

## JavaScript stored procedures: what the object actually is

**Concept.** A Snowflake JS SP is a **JavaScript object** stored in metadata. The SQL inside exists only as string literals passed to `snowflake.execute({sqlText: ...})` at runtime. No compiled SQL form of the procedure persists anywhere.

**The why.** Coming from T-SQL, the instinct is "an SP is SQL." In Snowflake, T-SQL's dual role (query language + procedural language) is split: SQL for queries, a general-purpose language (JS/Python/Java/Scala) for procedural control. Two-phase timeline: creation time (DDL stores the JS) vs. call time (JS runs, executes SQL strings one statement per `execute()` call).

**Production framing.** JS SPs are the older style, common in tutorials. **Snowflake Scripting** now allows SQL-native procedural SPs much closer to T-SQL. Practical dev workflow: write and test each SQL statement in a worksheet first, then wrap validated SQL in the JS shell. Inspect existing SPs with `GET_DDL()`, `SHOW PROCEDURES`, `INFORMATION_SCHEMA.PROCEDURES`.

**Interview one-liner.** "In a Snowflake JS proc the SQL is just runtime strings — the stored object is JavaScript, which is why each statement needs its own `snowflake.execute()` call."

## Streams + Tasks for CDC-style processing

**Concept.** A stream tracks changes on a table (exposing `METADATA$ACTION` etc.); a task runs SQL on a schedule/trigger. Together they drive incremental SCD1 MERGE patterns without external orchestration.

**Gotcha learned the hard way.** Filter `METADATA$ACTION = 'INSERT'` belongs in the `USING` subquery of the MERGE, not in the `WHEN MATCHED` / `WHEN NOT MATCHED` clauses — otherwise new records get silently skipped.

## Misc quick hits

- **Time Travel:** query historical state with `BEFORE (STATEMENT => '<query_id>')`; recover missing rows via LEFT JOIN from the snapshot.
- **COPY INTO load metadata:** Snowflake remembers loaded files (per file, ~64 days) and skips them; `FORCE = TRUE` overrides. `metadata$filename` and `metadata$file_row_number` give row-level provenance — cheap lineage worth adding to raw tables.
- **VARIANT/JSON:** extract with `JSON_DATA:field::TYPE`; arrays need `LATERAL FLATTEN` (chained for nested structures).
