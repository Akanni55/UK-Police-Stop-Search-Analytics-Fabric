# Validation and evidence

[Back to project](../README.md)

## Evidence basis

The following results were confirmed during interactive implementation and load testing. They are not results from a repository CI execution. Workspace screenshots and deployable exports remain requested; this page distinguishes those missing artefacts from the confirmed implementation results.

| Validation | Observed result |
|---|---|
| Raw request completion | 1,427 / 1,427 files after targeted recovery |
| File content | All planned responses valid arrays and non-empty |
| File-audit record-count sum vs Bronze | 1,436,597, difference 0 |
| Bronze vs Silver | 1,436,597 each, difference 0 |
| Silver vs Gold fact | 1,436,597 each, difference 0 |
| Loaded force/month coverage | 42 contributing forces, 36 publication months |
| Full timestamp conversion failures | 0 |
| Missing coordinate pairs | 160,768 records; about 11.2% |
| Invalid supplied coordinate ranges | 0 in the performed profiling |
| Missing normalised outcome information | 63,660 records; about 4.4% |
| Selected-scope reload | Metropolitan, August 2025; counts matched and global total unchanged |
| Processing parent | Silver and Gold completed successfully in sequence |
| Gold calendar | 1,462 rows including unknown, 2023-01-01 to 2026-12-31 |

The outcome count reflects the implemented normalisation, including its `false`-to-NULL limitation. It is not proof that all those source records omitted outcome information.

## Reproduce the count reconciliation

```sql
SELECT 'Bronze' AS layer, COUNT_BIG(*) AS rows_loaded
FROM [LH_Uk_Police_Crime].[bronze].[stopsearch]
UNION ALL
SELECT 'Silver', COUNT_BIG(*) FROM silver.StopSearch
UNION ALL
SELECT 'Gold', COUNT_BIG(*) FROM gold.FactStopSearch;
```

## Inspect a selected replacement scope

```sql
SELECT 'Silver' AS layer, COUNT_BIG(*) AS rows_loaded
FROM silver.StopSearch
WHERE force_id = 'metropolitan' AND source_month = '2025-08'
UNION ALL
SELECT 'Gold', COUNT_BIG(*)
FROM gold.FactStopSearch
WHERE force_id = 'metropolitan' AND source_month = '2025-08';
```

A matched total alone does not establish field-by-field equality. The implemented fact loader additionally rejects repeated source positions and unresolved dimension keys. Broader automated referential and semantic tests remain future work.

## Inspect quality flags

```sql
SELECT
    COUNT_BIG(*) AS total_rows,
    SUM(CAST(CASE WHEN has_valid_timestamp = 0 THEN 1 ELSE 0 END AS bigint))
        AS invalid_or_missing_timestamps,
    SUM(CAST(CASE WHEN has_outcome_information = 0 THEN 1 ELSE 0 END AS bigint))
        AS missing_normalised_outcome_information,
    SUM(CAST(CASE WHEN has_valid_coordinates = 0 THEN 1 ELSE 0 END AS bigint))
        AS missing_or_invalid_coordinates,
    SUM(CAST(CASE WHEN publication_month_matches_uk = 0 THEN 1 ELSE 0 END AS bigint))
        AS publication_month_mismatches
FROM silver.StopSearch;
```

A publication-month mismatch is an investigation flag, not an automatic error. It is not a quarantine criterion.

## Performance evidence boundary

The sequential historical extraction took many hours and required a four-request recovery. No controlled timing comparison, capacity measurement, parallelism experiment or before/after optimisation result has been captured. Scoped reprocessing avoids selecting the entire dataset for replacement, but a measured speedup is not claimed.

## Requested evidence pack

- Ingestion parent and child canvas screenshots with parameters visible.
- Reference pipeline and successful processing parent run.
- Silver and Gold child canvases and representative audit rows.
- Reconciliation query result and full semantic-model relationship screenshot.
- Final report screenshots and published URL when available.
- Sanitised notebook, pipeline and stored-procedure exports.

This repository intentionally does not embed unrelated screenshots or fabricate a report preview.
