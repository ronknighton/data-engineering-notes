# dbt

## dbt doesn't load data — layer ownership

**Concept.** dbt is the T in ELT. It only transforms data already in the warehouse. The raw/bronze layer is populated by something else (COPY INTO, Snowpipe, Fivetran, a Python/Glue job) and appears in dbt only as **sources** declared in a `sources.yml` — referenced and testable, never built.

**The why.** Explains why projects differ on where "bronze" lives: the cleaner pattern is that each layer is owned by the tool that writes it. Ingestion owns bronze; dbt owns silver/gold. A `raw` schema *configured inside dbt* blurs that boundary.

**Interview one-liner.** "Layer names are convention; layer ownership is the design decision — ingestion owns raw, dbt owns everything downstream."

## Medallion naming is convention, not architecture

Bronze/silver/gold (Databricks marketing term), raw/staging/mart (dbt style guide: staging → intermediate → marts), raw/transform/curated — all the same progressive-refinement idea. Translate fluently; note that "medallion" is the Databricks flavor of a vendor-neutral pattern.

## `generate_schema_name` macro

**Concept.** By default dbt builds custom schemas as `<target_schema>_<custom_schema>` (e.g., `dbt_rhk_silver`). The standard override macro makes `schema: silver` mean literally `SILVER`.

**The why.** The default is deliberate: it namespaces schemas per developer so teammates don't clobber each other in dev. Teams override it for production where literal names must match diagrams and grants. If a project lacks the macro, check Snowflake — models are landing in concatenated schemas.

**Interview one-liner.** "dbt's schema concatenation default is dev-environment namespacing; overriding `generate_schema_name` is a conscious prod decision, not boilerplate."

## Schema config: dbt_project.yml vs. model config

Per-folder `schema:` in `dbt_project.yml` and `{{ config(schema=...) }}` in a model do the identical thing. Folder-level config is preferred (DRY): organize models into layer folders, configure once.

## Pre-hooks and the `COPY INTO` macro pattern (SCD2 project)

**Concept.** A pre-hook is arbitrary SQL dbt runs before materializing a model. The SCD2 project wraps `DELETE` + `COPY INTO @stage` in a macro (`macros_copy_csv`) and invokes it as the silver model's pre-hook — so `dbt run` truncates and reloads the bronze work table, then builds the transform.

**The why.** It makes dbt orchestrate ingestion it normally wouldn't do. Works for a self-contained workshop; architecturally debatable because ingestion gets hidden inside a transformation tool where it's harder to monitor, retry, and schedule independently.

**Production framing.** Real pipelines: Snowpipe, an Airflow task, or a managed connector for ingestion; dbt starts at transform.

**Interview one-liner.** "Pre-hook loading works, but I'd use Snowpipe or an orchestrator in production so ingestion is independently observable and retryable."

## `vars`: configuration vs. parameters

**Concept.** dbt has three kinds of "inputs," and confusing them is easy coming from stored procedures:

| Mechanism | Scope | Analogy from SQL Server world |
|---|---|---|
| Macro arguments — `{% macro m(table_nm) %}` | Per call site | SP parameters |
| `vars` — `{{ var('stage_name') }}` | Per invocation, with project-level defaults in `dbt_project.yml` | Config table / SQLCMD variables |
| `env_var()` — `{{ env_var('SNOWFLAKE_PASSWORD') }}` | Machine/OS environment | Environment variables; keeps secrets out of version control |

**The why.** `vars` in `dbt_project.yml` are *defaults*, not hardcoded values. The caller can still override at invocation time:

```bash
dbt run --vars '{"stage_name": "MYDB.BRONZE.PROD_S3_STAGE", "purge_status": "TRUE"}'
```

Precedence: command-line `--vars` beats yml defaults. In production, the "caller" is an Airflow task or dbt Cloud job injecting environment-appropriate values. Values like stage name and raw database belong in vars (they change per *environment*, not per call); values like the target table name belong in macro arguments (they change per *call*).

**Deeper structural point.** There is no runtime call stack in dbt. `dbt run` compiles the entire project — every model, macro, and var reference — into static SQL up front, then executes the compiled DAG. Models relate through `ref()`/`source()`, which are compile-time dependency declarations, not runtime calls. All inputs are resolved before any SQL reaches Snowflake.

**Interview one-liner.** "dbt vars are invocation-scoped configuration with project-level defaults — yml sets the default, the caller (CLI, Airflow, dbt Cloud job) overrides at run time, and macro arguments handle true per-call parameters."

## Snapshots = built-in SCD Type 2

**Concept.** `{% snapshot %}` blocks compare current rows (by `unique_key`) against the snapshot table each `dbt snapshot` run. Changed rows: old version closed out (`DBT_VALID_TO` set), new version inserted (`DBT_VALID_FROM`). No hand-written MERGE.

**Strategy choice (interview-relevant).** `strategy='check'` with `check_cols` compares column values — robust, costs comparison work. `strategy='timestamp'` trusts an `updated_at` column — cheaper, only valid if the source reliably maintains it.

**Presentation layer.** Gold view renames `DBT_VALID_FROM/TO` to version start/end, coalesces open-ended NULL valid-to into the `9999-12-31` sentinel. Current-row filter: `DBT_VALID_TO IS NULL` (or sentinel equality downstream).

**Interview one-liner.** "dbt snapshots give you SCD2 for free; the real decision is check vs. timestamp strategy, which trades compute against trust in the source's updated_at."

## `set_query_tag` macro

Runs `ALTER SESSION SET QUERY_TAG = '<model_name>'` before each model, so Snowflake `QUERY_HISTORY` attributes every query — and its compute cost — to a dbt model. Pure observability; strong cost-consciousness signal.

## dbt Fusion 2.0 / platform changes (as of mid-2026)

- **Test arguments schema change (dbt1159):** generic test parameters must nest under `arguments:` — `accepted_values: → arguments: → values: [...]`. Separates test *config* (severity, where) from test *arguments*. Zero-arg tests (`unique`, `not_null`) unaffected.
- **Connection Profiles:** dbt platform bundles connection details + deployment credentials into a reusable object attached to environments (replaces per-environment inline settings shown in older videos).
- **Key pair auth mandatory:** new Snowflake credentials in dbt platform require an RSA key pair (`ALTER USER ... SET RSA_PUBLIC_KEY=...`); username/password no longer accepted.

## yml conventions

- One `schema.yml` per folder, colocated with the models it describes.
- In a `sources:` block, `name:` is the **source name** (first arg of `{{ source('name','table') }}`), not the model name — mismatching breaks the ref.
- `version: 2` in schema files vs. `config-version: 2` in dbt_project.yml are unrelated version fields; modern dbt (1.5+) allows omitting `version:` in some contexts.
