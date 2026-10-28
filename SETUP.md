# Setup and reproduction guide

[Back to project](README.md)

## Read this first

This repository currently provides a technical case study and implementation walkthroughs with inline code. It is **not yet a one-click deployment package**. The exact deployed notebook, pipeline and SQL object exports have been requested and will be added after verification. Code excerpts are labelled where they omit surrounding implementation. Do not mistake illustrative fragments for a complete schema deployment.

## Prerequisites

- A Fabric workspace backed by suitable capacity and permissions to create Lakehouse, Warehouse, notebooks and pipelines.
- Permission to create Warehouse schemas, tables and stored procedures, and read the Lakehouse SQL endpoint.
- Public API connectivity and an appropriate organisational authentication connection for Fabric activities.
- Power BI semantic-model/report creation permissions.

Use your own workspace and item identifiers. No tenant-specific connection identifiers or credentials are required in this repository.

## Build order

1. Create `LH_Uk_Police_Crime` and `WH_Police_Analytics` in the same workspace.
2. Configure the REST connection using `https://data.police.uk/api/`.
3. Retrieve forces and availability metadata; create Bronze references and the request plan.
4. Validate a small sample of force/month requests before scaling the plan. This is an implementation test, not a regional scope restriction.
5. Run historical API ingestion, retain JSON files and recover failed pairs only.
6. Validate file content and counts; load Bronze with source-file and array-position lineage.
7. Confirm the endpoint's actual table names and columns before compiling Warehouse procedures.
8. Create `silver`, `gold` and `audit` schemas and the tables described in the dictionary.
9. Create the Silver loader and audit wrapper; reconcile the full Silver load.
10. Create and load Date, Force and the three combination dimensions; create the fact and reconcile its initial load.
11. Configure Silver and Gold pipelines, then the processing parent; test a selected scope.
12. Create `SM_Police_StopSearch` over the six Gold tables, add relationships and DAX measures.
13. Build the report and verify totals before publishing it as **UK Police Stop and Search Insights**.

See [ingestion](ingestion/api-to-bronze.md), [Silver](implementation/silver.md), [Gold](implementation/gold.md) and [operations](monitoring/operations.md) for the code and reasoning used at each boundary.

## Verify source visibility before loading

From the Lakehouse SQL endpoint:

```sql
SELECT DB_NAME() AS database_name, TABLE_SCHEMA, TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME LIKE '%stop%search%'
ORDER BY TABLE_SCHEMA, TABLE_NAME;
```

Then, from Warehouse:

```sql
SELECT COUNT_BIG(*) AS source_rows
FROM [LH_Uk_Police_Crime].[bronze].[stopsearch];
```

Resolve access and naming before running a replacement load. The SQL endpoint and Warehouse are separate surfaces; issuing table creation against the wrong item produced a policy error during development.

## Execute a selected processing scope

Use the parent pipeline parameters:

```text
p_force_id = metropolitan
p_source_month = 2025-08
```

These values must exist in your loaded Bronze snapshot. The developed loaders reject an empty selected source rather than deleting existing target data. Empty parent parameters request full-scope processing; they do not automatically discover or download new source months.

## Reproduction boundaries to resolve

- Current API availability may differ from this captured snapshot; discover available pairs dynamically.
- Notebook schema support and attached item context must match your environment.
- Verify Bronze write mode before replaying a notebook.
- Serialise the processing parent because dimension key allocation is not concurrency-safe.
- Refine the false-outcome mapping before treating outcome recording as strict source completeness.
- Configure and validate model refresh behaviour after Gold updates.
- Add CI, environment promotion, shared run correlation and scheduling only when actually implemented.
