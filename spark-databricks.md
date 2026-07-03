# Spark / Databricks

*(Thinner than the other files — backfill as Databricks work resumes. Known gap: DEA covers Databricks but the System Design Playbook doesn't, and Delta Lake / Unity Catalog come up at non-AWS-native companies.)*

## UDF landscape

Standard (Python) UDFs are slow (row-at-a-time serialization); Pandas UDFs vectorize via Arrow; `mapPartitions` / `foreachPartition` amortize per-partition setup (e.g., DB connections). Default answer: use built-in Spark SQL functions first, UDFs last.

## Data skew and salting

Skewed keys concentrate work on one executor. Salting appends a random suffix to hot keys to spread them across partitions, then re-aggregates. Interview staple for "my join is slow" scenarios.

## Delta Lake

Open table format adding ACID transactions, time travel, and schema enforcement on top of Parquet in object storage — the thing that makes a data lake behave like a warehouse ("lakehouse"). Created by Databricks but open source; cloning (shallow/deep) supports dev/test copies.

## Medallion architecture

Bronze/silver/gold is Databricks' framing of progressive refinement (see dbt.md — naming is convention). Delta Live Tables automates the medallion pipeline; worth naming as the managed step up from hand-orchestrated Spark jobs.

## Unity Catalog

Databricks' governance layer: centralized catalog, lineage, access control across workspaces. **Study gap flagged for interview prep** — build a mental model of Lakehouse = Delta Lake + Unity Catalog for interviews outside the AWS/Snowflake bubble.
