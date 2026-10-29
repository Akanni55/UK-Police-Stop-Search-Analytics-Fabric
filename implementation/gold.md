# Gold: dimensions, stable keys and a source-record fact

[Back to project](../README.md)

## Model and load order

| Order | Procedure | Target |
|---|---|---|
| 1 | `gold.usp_LoadDimDate` | Continuous full-year calendar and unknown member |
| 2 | `gold.usp_LoadDimForce` | Reference forces, Silver-only fallback IDs and unknown member |
| 3 | `gold.usp_LoadProfileDimensions` | Demographic, search-profile and outcome combinations |
| 4 | `gold.usp_RunFactStopSearchLoad` | Audited call to `gold.usp_LoadFactStopSearch` |

The key-based relationships are shown in the [architecture](../architecture/architecture-overview.md). Code below explains the implemented patterns inline. Full definitions are now available in the [complete procedure reconstruction](complete-procedures.md). The excerpts on this page explain the design; the complete page clearly distinguishes reconstructed code from live exports.

## Date dimension

Dates span full calendar years from the earliest to latest valid UK search dates. A four-digit Cartesian number generator supplies up to 10,000 day offsets. A range guard rejects longer spans. `date_key` is the integer `YYYYMMDD`; the extra key 0 has no real calendar date.

```sql
CONVERT(int, CONVERT(varchar(8), calendar_date, 112)) AS date_key
```

Weekday calculation is independent of `DATEFIRST`. January 1900 provides a known Monday anchor; the double modulo also handles earlier dates:

```sql
((DATEDIFF(DAY, CAST('19000101' AS date), calendar_date) % 7 + 7) % 7) + 1
```

The current calendar covers 2023–2026: 1,461 actual dates plus one unknown member. A conventional Power BI date-table designation over this column needs special treatment because the unknown row is NULL; do not claim it is marked as a date table. The initial measures do not require classic time-intelligence functions.

## Force dimension

`DimForce` includes the full captured reference list, even when a force has no loaded records in the period. A force found only in Silver is retained with its ID as the fallback label. Names are overwritten from reference metadata while surrogate keys remain unchanged.

```sql
UPDATE target
SET force_name = COALESCE(NULLIF(TRIM(source.name), ''), source.id)
FROM gold.DimForce AS target
INNER JOIN [LH_Uk_Police_Crime].[bronze].[forces] AS source
    ON target.force_id = source.id
WHERE target.force_key <> 0;
```

## Combination dimensions

The profiled source had 2,111 demographic combinations, 398 search profiles and 8 outcome combinations. `DimDemographic` combines age range, gender and both ethnicity classifications. `DimSearchProfile` combines search type, legislation and object. `DimOutcome` retains top-level and nested outcome values.

Distinct combinations already in the target are excluded. New rows use the current maximum key plus a window-generated offset:

```sql
-- Excerpt: NewCombinations has already excluded existing members.
SELECT
    @MaxSearchProfileKey
      + CAST(ROW_NUMBER() OVER (
          ORDER BY search_type, legislation, object_of_search
        ) AS int) AS search_profile_key,
    search_type, legislation, object_of_search
FROM NewCombinations;
```

This allocation is safe only under the documented serial execution discipline. Run one processing parent at a time. Schedules and manual runs must not overlap.

## Preserve the source grain

`gold.FactStopSearch` stores dimension keys, event timestamps, coordinates, operational flags, publication metadata and lineage. `search_count` is 1. There is no claimed unique-person identifier and no global source-event surrogate generated from descriptive attributes.

Before replacement, the loader rejects duplicate `(source_file, source_record_index)` positions within the selected Silver scope:

```sql
SELECT source_file, source_record_index
FROM silver.StopSearch
WHERE (@ForceId IS NULL OR force_id = @ForceId)
  AND (@SourceMonth IS NULL OR source_month = @SourceMonth)
GROUP BY source_file, source_record_index
HAVING COUNT_BIG(*) > 1;
```

This check prevents accidentally repeated ingestion rows. It does not remove separate source records with the same age, place, time or outcome.

## Null-safe dimension resolution

A missing attribute should match the corresponding missing attribute in a stored combination. Plain equality does not match two NULLs. The fact loader uses the supported `IS NOT DISTINCT FROM` predicate:

```sql
FROM silver.StopSearch AS s
LEFT JOIN gold.DimSearchProfile AS sp
    ON sp.search_type IS NOT DISTINCT FROM s.search_type
   AND sp.legislation IS NOT DISTINCT FROM s.legislation
   AND sp.object_of_search IS NOT DISTINCT FROM s.object_of_search
```

Demographic and outcome joins follow the same pattern. The original combination loaders use equivalent explicit `a = b OR (a IS NULL AND b IS NULL)` comparisons.

A wholly missing combination maps to key 0; a partially missing combination has its own member. A missing dimension row is not silently converted to unknown: fact dimension-key columns are NOT NULL, so an unresolved match fails the insert. Reconciliation rejects extra rows if duplicate dimension combinations multiply fact rows. Dimension uniqueness also merits explicit checks; keys are not enforced database constraints in the current DDL.

## Scope replacement and audit

The fact loader accepts force/month parameters, deletes only that scope and inserts its complete replacement in a transaction. Silver and fact row counts must match before commit. The first full load reconciled 1,436,597 rows. The selected Metropolitan/August 2025 rerun left the global count unchanged.

```sql
EXEC gold.usp_RunFactStopSearchLoad
    @ForceId = 'metropolitan',
    @SourceMonth = '2025-08',
    @PipelineRunId = 'manual-example';
```

The wrapper records fact-stage audit status. Dimension activities are monitored in Fabric, not written to the fact audit table. Missing dimension matches, duplicate source positions and zero-source replacement requests are load errors, not instructions to delete existing target data.
