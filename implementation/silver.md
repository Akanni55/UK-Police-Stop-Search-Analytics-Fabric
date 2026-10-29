# SQL Silver: parsing, quality and scoped replacement

[Back to project](../README.md)

## Why SQL is used here

Raw JSON and Bronze Delta data stay in the Lakehouse. Writable transformations execute in `WH_Police_Analytics`, reading the exact source object:

```sql
SELECT COUNT_BIG(*) AS bronze_rows
FROM [LH_Uk_Police_Crime].[bronze].[stopsearch];
```

The lowercase table name matters in the implemented environment. An earlier `StopSearch` reference failed to resolve. The repair was to inspect `INFORMATION_SCHEMA.TABLES`, use the actual identifier and roll back the open transaction before retrying.

The snippets below are implementation excerpts and explanatory reductions of the developed procedures, not a claim that an arbitrary excerpt can replace the full deployed procedure. All seven full definitions are now provided as [clearly labelled reconstructions](complete-procedures.md). They are not live Warehouse exports.

## Parse once, convert deliberately

`silver.usp_LoadStopSearch` extracts nested JSON properties with `JSON_VALUE`, trims blank text, parses types with `TRY_CONVERT` and uses explicit true/false-to-bit mapping.

```sql
WITH Extracted AS
(
    SELECT
        force_id, source_month, source_file, source_record_index,
        NULLIF(TRIM(JSON_VALUE(raw_payload, '$.datetime')), '')
            AS source_datetime,
        NULLIF(TRIM(JSON_VALUE(raw_payload, '$.type')), '')
            AS search_type,
        JSON_VALUE(raw_payload, '$.location.latitude') AS latitude_text,
        JSON_VALUE(raw_payload, '$.location.longitude') AS longitude_text,
        JSON_VALUE(raw_payload, '$.involved_person') AS involved_person_text
    FROM [LH_Uk_Police_Crime].[bronze].[stopsearch]
),
Converted AS
(
    SELECT *,
        TRY_CONVERT(datetimeoffset(6), source_datetime) AS parsed_datetime,
        TRY_CONVERT(decimal(12,8), latitude_text) AS latitude,
        TRY_CONVERT(decimal(12,8), longitude_text) AS longitude
    FROM Extracted
)
SELECT
    force_id, source_month, source_file, source_record_index,
    source_datetime, search_type,
    CAST(parsed_datetime AT TIME ZONE 'UTC' AS datetime2(6))
        AS search_datetime_utc,
    CAST(parsed_datetime AT TIME ZONE 'GMT Standard Time' AS datetime2(6))
        AS search_datetime_uk,
    latitude, longitude,
    CASE LOWER(involved_person_text)
        WHEN 'true' THEN CAST(1 AS bit)
        WHEN 'false' THEN CAST(0 AS bit)
        ELSE NULL
    END AS involved_person
FROM Converted;
```

The deployed insert also includes demographic fields, legislation, search object, outcomes, street information, operation details and quality flags listed in the [dictionary](../docs/data-dictionary.md).

## Publication month is not event month

The response month is retained as `source_month`; it is not overwritten with a date derived from the event timestamp. `search_date_uk` comes from the timezone-adjusted UK timestamp.

An observed source timestamp of `2023-08-31T23:00:00+00:00` falls on 1 September in UK summer time. The project's latest UK timestamp extends into 1 September 2026 even though the last publication month is August. Neither boundary automatically justifies deleting a record.

## Quality flags preserve records

```sql
CAST(CASE
    WHEN latitude BETWEEN -90 AND 90
     AND longitude BETWEEN -180 AND 180
    THEN 1 ELSE 0
END AS bit) AS has_valid_coordinates
```

This is a numeric range check, not a check that a point falls inside a force boundary or represents an exact search location. Missing coordinates remain NULL. Missing timestamps and outcomes receive flags; rows are not quarantined or removed by these rules.

### Important outcome caveat

The implemented loader normalises empty outcome text and the literal `false` to NULL:

```sql
NULLIF(NULLIF(TRIM(JSON_VALUE(raw_payload, '$.outcome')), ''), 'false')
```

The API dictionary distinguishes a false outcome from an unprovided value. The current normalisation therefore loses that distinction in Silver unless nested outcome data provides information. Bronze retains the original payload. See [limitations](../docs/limitations.md) before interpreting the outcome-recording measure as strict source completeness; refining this mapping and reprocessing the affected scopes is a planned correction.

## Scoped, repeatable processing

The procedure accepts optional `@ForceId` and `@SourceMonth`. Both NULL means all current Bronze records. A selected scope is deleted and rebuilt inside a transaction, and its output count must match its source count.

```sql
-- Predicate used consistently for source selection, deletion and reconciliation:
WHERE (@ForceId IS NULL OR force_id = @ForceId)
  AND (@SourceMonth IS NULL OR source_month = @SourceMonth)
```

The transaction pattern is:

```sql
BEGIN TRY
    BEGIN TRANSACTION;
    -- Count source; reject an empty selected source.
    -- Delete selected target scope.
    -- Insert every selected source record after transformation.
    -- Throw if source and target counts differ.
    COMMIT TRANSACTION;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
    THROW;
END CATCH;
```

These are replacement loads, not CDC or automatic change detection. No event-attribute deduplication occurs.

## Audited execution

The pipeline calls `silver.usp_RunStopSearchLoad`, which inserts a Running audit row, calls the loader and records success or failure. A failure is rethrown so the activity fails visibly.

```sql
EXEC silver.usp_RunStopSearchLoad
    @ForceId = 'metropolitan',
    @SourceMonth = '2025-08',
    @PipelineRunId = 'manual-example';
```

The wrapper runs without an enclosing transaction so audit updates can survive a load rollback. The outer caller can catch errors occurring inside the called procedure, including a name-resolution failure that bypasses the loader's own same-level catch.
