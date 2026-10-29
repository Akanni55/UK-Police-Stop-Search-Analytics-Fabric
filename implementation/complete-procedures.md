# Complete SQL procedure reconstructions

[Back to project](../README.md)

## Provenance

These seven complete definitions reconstruct the SQL implementation developed in the project conversation. They are **not current procedure bodies extracted from the Warehouse**. Procedure existence and successful runs are evidenced separately. Formatting, local variable names and error text have been consolidated; the reconstruction is not asserted to be byte-identical to deployed definitions. No Fabric object is modified by publishing these pages.

Tables and column types must already match the [data dictionary](../docs/data-dictionary.md). Run each CREATE OR ALTER definition as a separate query batch in a test Warehouse first. Do not redeploy to the working environment merely for documentation. This page supplies full procedure code inline without requiring readers to open separate script files.

The existing false-outcome normalisation is retained and documented as a limitation, rather than quietly changing outcome semantics during reconstruction. Scope parameters are normalised to NULL when blank. Dimension key allocation remains serial. Audit post-processing is outside the inner data transaction; a post-commit audit failure does not undo a successful data load.

## silver.usp_LoadStopSearch

```sql
CREATE OR ALTER PROCEDURE silver.usp_LoadStopSearch
    @ForceId varchar(100) = NULL,
    @SourceMonth varchar(7) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @ExpectedRows bigint, @LoadedRows bigint;
    DECLARE @LoadedAt datetime2(6) = SYSUTCDATETIME();
    SET @ForceId = NULLIF(TRIM(@ForceId), '');
    SET @SourceMonth = NULLIF(TRIM(@SourceMonth), '');
    BEGIN TRY
        BEGIN TRANSACTION;
        SELECT @ExpectedRows = COUNT_BIG(*)
        FROM [LH_Uk_Police_Crime].[bronze].[stopsearch]
        WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        IF @ExpectedRows = 0 THROW 50001, 'No Bronze records; target unchanged.', 1;
        DELETE FROM silver.StopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        ;WITH Extracted AS
        (
            SELECT force_id, source_month, source_file, source_record_index,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.datetime')), '') AS source_datetime,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.type')), '') AS search_type,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.legislation')), '') AS legislation,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.object_of_search')), '') AS object_of_search,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.age_range')), '') AS age_range,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.gender')), '') AS gender,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.self_defined_ethnicity')), '') AS self_defined_ethnicity,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.officer_defined_ethnicity')), '') AS officer_defined_ethnicity,
                NULLIF(NULLIF(TRIM(JSON_VALUE(raw_payload, '$.outcome')), ''), 'false') AS outcome,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.outcome_object.id')), '') AS outcome_code,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.outcome_object.name')), '') AS outcome_name,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.location.street.name')), '') AS street_name,
                NULLIF(TRIM(JSON_VALUE(raw_payload, '$.operation_name')), '') AS operation_name,
                CASE LOWER(JSON_VALUE(raw_payload, '$.involved_person')) WHEN 'true' THEN CAST(1 AS bit) WHEN 'false' THEN CAST(0 AS bit) ELSE NULL END AS involved_person,
                CASE LOWER(JSON_VALUE(raw_payload, '$.outcome_linked_to_object_of_search')) WHEN 'true' THEN CAST(1 AS bit) WHEN 'false' THEN CAST(0 AS bit) ELSE NULL END AS outcome_linked_to_search,
                CASE LOWER(JSON_VALUE(raw_payload, '$.operation')) WHEN 'true' THEN CAST(1 AS bit) WHEN 'false' THEN CAST(0 AS bit) ELSE NULL END AS operation,
                CASE LOWER(JSON_VALUE(raw_payload, '$.removal_of_more_than_outer_clothing')) WHEN 'true' THEN CAST(1 AS bit) WHEN 'false' THEN CAST(0 AS bit) ELSE NULL END AS removal_of_outer_clothing,
                TRY_CONVERT(decimal(12,8), JSON_VALUE(raw_payload, '$.location.latitude')) AS latitude,
                TRY_CONVERT(decimal(12,8), JSON_VALUE(raw_payload, '$.location.longitude')) AS longitude,
                TRY_CONVERT(bigint, JSON_VALUE(raw_payload, '$.location.street.id')) AS street_id
            FROM [LH_Uk_Police_Crime].[bronze].[stopsearch]
            WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth)
        ), Converted AS
        (
            SELECT *, TRY_CONVERT(datetimeoffset(6), source_datetime) AS parsed_datetime
            FROM Extracted
        ), Localised AS
        (
            SELECT *,
                CAST(parsed_datetime AT TIME ZONE 'UTC' AS datetime2(6)) AS search_datetime_utc,
                CAST(parsed_datetime AT TIME ZONE 'GMT Standard Time' AS datetime2(6)) AS search_datetime_uk
            FROM Converted
        )
        INSERT INTO silver.StopSearch
        (force_id, source_month, source_file, source_record_index, source_datetime, search_datetime_utc, search_datetime_uk, search_date_uk, search_type, involved_person, legislation, object_of_search, age_range, gender, self_defined_ethnicity, officer_defined_ethnicity, outcome, outcome_code, outcome_name, outcome_linked_to_search, latitude, longitude, street_id, street_name, operation, operation_name, removal_of_outer_clothing, has_outcome_information, has_valid_coordinates, has_valid_timestamp, publication_month_matches_uk, silver_loaded_at)
        SELECT force_id,
            source_month,
            source_file,
            source_record_index,
            source_datetime,
            search_datetime_utc,
            search_datetime_uk,
            CAST(search_datetime_uk AS date),
            search_type,
            involved_person,
            legislation,
            object_of_search,
            age_range,
            gender,
            self_defined_ethnicity,
            officer_defined_ethnicity,
            outcome,
            outcome_code,
            outcome_name,
            outcome_linked_to_search,
            latitude,
            longitude,
            street_id,
            street_name,
            operation,
            operation_name,
            removal_of_outer_clothing,
            CAST(CASE WHEN outcome IS NOT NULL OR outcome_code IS NOT NULL OR outcome_name IS NOT NULL THEN 1 ELSE 0 END AS bit),
            CAST(CASE WHEN latitude BETWEEN -90 AND 90 AND longitude BETWEEN -180 AND 180 THEN 1 ELSE 0 END AS bit),
            CAST(CASE WHEN parsed_datetime IS NOT NULL THEN 1 ELSE 0 END AS bit),
            CAST(CASE WHEN source_month = LEFT(CONVERT(varchar(10), search_datetime_uk, 23), 7) THEN 1 ELSE 0 END AS bit),
            @LoadedAt
        FROM Localised;
        SELECT @LoadedRows = COUNT_BIG(*) FROM silver.StopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        IF @LoadedRows <> @ExpectedRows THROW 50002, 'Bronze/Silver count mismatch.', 1;
        COMMIT TRANSACTION;
        SELECT 'Succeeded' AS load_status, @ExpectedRows AS bronze_rows,
            @LoadedRows AS silver_rows, @LoadedRows - @ExpectedRows AS row_difference;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## silver.usp_RunStopSearchLoad

```sql
CREATE OR ALTER PROCEDURE silver.usp_RunStopSearchLoad
    @ForceId varchar(100) = NULL,
    @SourceMonth varchar(7) = NULL,
    @PipelineRunId varchar(100) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    IF @@TRANCOUNT > 0 THROW 50030, 'Run without an existing transaction.', 1;
    DECLARE @LoadId varchar(36) = CONVERT(varchar(36), NEWID());
    DECLARE @SourceRows bigint, @TargetRows bigint, @ErrorMessage varchar(8000);
    SET @ForceId = NULLIF(TRIM(@ForceId), '');
    SET @SourceMonth = NULLIF(TRIM(@SourceMonth), '');
    INSERT INTO audit.StopSearchLoad
        (load_id, pipeline_run_id, force_id, source_month, started_at_utc, load_status)
    VALUES (@LoadId, @PipelineRunId, @ForceId, @SourceMonth, SYSUTCDATETIME(), 'Running');
    BEGIN TRY
        EXEC silver.usp_LoadStopSearch @ForceId = @ForceId, @SourceMonth = @SourceMonth;
        SELECT @SourceRows = COUNT_BIG(*) FROM [LH_Uk_Police_Crime].[bronze].[stopsearch] WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        SELECT @TargetRows = COUNT_BIG(*) FROM silver.StopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        UPDATE audit.StopSearchLoad
        SET completed_at_utc = SYSUTCDATETIME(), bronze_rows = @SourceRows,
            silver_rows = @TargetRows, load_status = 'Succeeded'
        WHERE load_id = @LoadId;
    END TRY
    BEGIN CATCH
        SET @ErrorMessage = CONCAT('Error ', ERROR_NUMBER(), ': ', ERROR_MESSAGE());
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        UPDATE audit.StopSearchLoad
        SET completed_at_utc = SYSUTCDATETIME(), load_status = 'Failed', error_message = @ErrorMessage
        WHERE load_id = @LoadId;
        THROW;
    END CATCH;
END;
```

## gold.usp_LoadDimDate

```sql
CREATE OR ALTER PROCEDURE gold.usp_LoadDimDate
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MinDate date, @MaxDate date, @StartDate date, @EndDate date;
    SELECT @MinDate = MIN(search_date_uk), @MaxDate = MAX(search_date_uk)
    FROM silver.StopSearch WHERE has_valid_timestamp = 1;
    IF @MinDate IS NULL OR @MaxDate IS NULL THROW 50010, 'No valid Silver dates.', 1;
    SET @StartDate = DATEFROMPARTS(YEAR(@MinDate), 1, 1);
    SET @EndDate = DATEFROMPARTS(YEAR(@MaxDate), 12, 31);
    IF DATEDIFF(DAY, @StartDate, @EndDate) > 9999 THROW 50011, 'Calendar range too large.', 1;
    BEGIN TRY
        BEGIN TRANSACTION;
        IF NOT EXISTS (SELECT 1 FROM gold.DimDate WHERE date_key = 0)
            INSERT INTO gold.DimDate (date_key, day_name) VALUES (0, 'Unknown');
        ;WITH Digits AS
        (
            SELECT 0 AS n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3
            UNION ALL SELECT 4 UNION ALL SELECT 5 UNION ALL SELECT 6
            UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9
        ), Numbers AS
        (
            SELECT a.n + b.n*10 + c.n*100 + d.n*1000 AS n
            FROM Digits a CROSS JOIN Digits b CROSS JOIN Digits c CROSS JOIN Digits d
        ), Calendar AS
        (
            SELECT DATEADD(DAY, n, @StartDate) AS calendar_date
            FROM Numbers WHERE n <= DATEDIFF(DAY, @StartDate, @EndDate)
        ), Attributes AS
        (
            SELECT calendar_date,
                CONVERT(int, CONVERT(varchar(8), calendar_date, 112)) AS date_key,
                ((DATEDIFF(DAY, CAST('19000101' AS date), calendar_date)%7+7)%7)+1 AS weekday_number
            FROM Calendar
        )
        INSERT INTO gold.DimDate
        (date_key, calendar_date, calendar_year, calendar_quarter, month_number,
         month_name, year_month, day_of_month, day_of_week_number, day_name, is_weekend)
        SELECT a.date_key, a.calendar_date, YEAR(a.calendar_date),
            DATEPART(QUARTER, a.calendar_date), MONTH(a.calendar_date),
            CASE MONTH(a.calendar_date)
                WHEN 1 THEN 'January' WHEN 2 THEN 'February' WHEN 3 THEN 'March'
                WHEN 4 THEN 'April' WHEN 5 THEN 'May' WHEN 6 THEN 'June'
                WHEN 7 THEN 'July' WHEN 8 THEN 'August' WHEN 9 THEN 'September'
                WHEN 10 THEN 'October' WHEN 11 THEN 'November' WHEN 12 THEN 'December' END,
            LEFT(CONVERT(varchar(10), a.calendar_date, 23), 7), DAY(a.calendar_date),
            a.weekday_number,
            CASE a.weekday_number WHEN 1 THEN 'Monday' WHEN 2 THEN 'Tuesday'
                WHEN 3 THEN 'Wednesday' WHEN 4 THEN 'Thursday' WHEN 5 THEN 'Friday'
                WHEN 6 THEN 'Saturday' WHEN 7 THEN 'Sunday' END,
            CAST(CASE WHEN a.weekday_number IN (6,7) THEN 1 ELSE 0 END AS bit)
        FROM Attributes a
        WHERE NOT EXISTS (SELECT 1 FROM gold.DimDate d WHERE d.date_key = a.date_key);
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## gold.usp_LoadDimForce

```sql
CREATE OR ALTER PROCEDURE gold.usp_LoadDimForce
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MaxForceKey int;
    BEGIN TRY
        BEGIN TRANSACTION;
        IF NOT EXISTS (SELECT 1 FROM gold.DimForce WHERE force_key = 0)
            INSERT INTO gold.DimForce (force_key, force_id, force_name) VALUES (0, NULL, 'Unknown');
        UPDATE target SET force_name = COALESCE(NULLIF(TRIM(source.name), ''), source.id)
        FROM gold.DimForce target
        INNER JOIN [LH_Uk_Police_Crime].[bronze].[forces] source ON target.force_id = source.id
        WHERE target.force_key <> 0;
        SELECT @MaxForceKey = COALESCE(MAX(force_key),0) FROM gold.DimForce;
        ;WITH ReferenceForces AS
        (
            SELECT id AS force_id, MAX(COALESCE(NULLIF(TRIM(name), ''), id)) AS force_name
            FROM [LH_Uk_Police_Crime].[bronze].[forces]
            WHERE NULLIF(TRIM(id), '') IS NOT NULL GROUP BY id
        ), SourceForces AS
        (
            SELECT force_id, force_name FROM ReferenceForces
            UNION ALL
            SELECT DISTINCT s.force_id, s.force_id FROM silver.StopSearch s
            WHERE NULLIF(TRIM(s.force_id), '') IS NOT NULL
              AND NOT EXISTS (SELECT 1 FROM ReferenceForces r WHERE r.force_id = s.force_id)
        ), NewForces AS
        (
            SELECT s.* FROM SourceForces s
            WHERE NOT EXISTS (SELECT 1 FROM gold.DimForce d WHERE d.force_id = s.force_id)
        )
        INSERT INTO gold.DimForce (force_key, force_id, force_name)
        SELECT @MaxForceKey + CAST(ROW_NUMBER() OVER (ORDER BY force_id) AS int),
            force_id, force_name FROM NewForces;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## gold.usp_LoadProfileDimensions

```sql
CREATE OR ALTER PROCEDURE gold.usp_LoadProfileDimensions
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MaxKey int;
    BEGIN TRY
        BEGIN TRANSACTION;
        IF NOT EXISTS (SELECT 1 FROM gold.DimDemographic WHERE demographic_key = 0)
            INSERT INTO gold.DimDemographic (demographic_key) VALUES (0);
        SELECT @MaxKey = COALESCE(MAX(demographic_key),0) FROM gold.DimDemographic;
        ;WITH SourceCombinations AS
        (
            SELECT DISTINCT age_range, gender, self_defined_ethnicity, officer_defined_ethnicity FROM silver.StopSearch
        ), NewCombinations AS
        (
            SELECT s.* FROM SourceCombinations s
            WHERE NOT EXISTS (SELECT 1 FROM gold.DimDemographic d WHERE (d.age_range = s.age_range OR (d.age_range IS NULL AND s.age_range IS NULL))
                  AND (d.gender = s.gender OR (d.gender IS NULL AND s.gender IS NULL))
                  AND (d.self_defined_ethnicity = s.self_defined_ethnicity OR (d.self_defined_ethnicity IS NULL AND s.self_defined_ethnicity IS NULL))
                  AND (d.officer_defined_ethnicity = s.officer_defined_ethnicity OR (d.officer_defined_ethnicity IS NULL AND s.officer_defined_ethnicity IS NULL)))
        )
        INSERT INTO gold.DimDemographic (demographic_key, age_range, gender, self_defined_ethnicity, officer_defined_ethnicity)
        SELECT @MaxKey + CAST(ROW_NUMBER() OVER (ORDER BY age_range, gender, self_defined_ethnicity, officer_defined_ethnicity) AS int),
            age_range, gender, self_defined_ethnicity, officer_defined_ethnicity
        FROM NewCombinations;

        IF NOT EXISTS (SELECT 1 FROM gold.DimSearchProfile WHERE search_profile_key = 0)
            INSERT INTO gold.DimSearchProfile (search_profile_key) VALUES (0);
        SELECT @MaxKey = COALESCE(MAX(search_profile_key),0) FROM gold.DimSearchProfile;
        ;WITH SourceCombinations AS
        (
            SELECT DISTINCT search_type, legislation, object_of_search FROM silver.StopSearch
        ), NewCombinations AS
        (
            SELECT s.* FROM SourceCombinations s
            WHERE NOT EXISTS (SELECT 1 FROM gold.DimSearchProfile d WHERE (d.search_type = s.search_type OR (d.search_type IS NULL AND s.search_type IS NULL))
                  AND (d.legislation = s.legislation OR (d.legislation IS NULL AND s.legislation IS NULL))
                  AND (d.object_of_search = s.object_of_search OR (d.object_of_search IS NULL AND s.object_of_search IS NULL)))
        )
        INSERT INTO gold.DimSearchProfile (search_profile_key, search_type, legislation, object_of_search)
        SELECT @MaxKey + CAST(ROW_NUMBER() OVER (ORDER BY search_type, legislation, object_of_search) AS int),
            search_type, legislation, object_of_search
        FROM NewCombinations;

        IF NOT EXISTS (SELECT 1 FROM gold.DimOutcome WHERE outcome_key = 0)
            INSERT INTO gold.DimOutcome (outcome_key, outcome_display_name, has_outcome_information) VALUES (0, 'Not recorded', 0);
        SELECT @MaxKey = COALESCE(MAX(outcome_key),0) FROM gold.DimOutcome;
        ;WITH SourceCombinations AS
        (
            SELECT DISTINCT outcome, outcome_code, outcome_name FROM silver.StopSearch
        ), NewCombinations AS
        (
            SELECT s.* FROM SourceCombinations s
            WHERE NOT EXISTS (SELECT 1 FROM gold.DimOutcome d WHERE (d.outcome = s.outcome OR (d.outcome IS NULL AND s.outcome IS NULL))
                  AND (d.outcome_code = s.outcome_code OR (d.outcome_code IS NULL AND s.outcome_code IS NULL))
                  AND (d.outcome_name = s.outcome_name OR (d.outcome_name IS NULL AND s.outcome_name IS NULL)))
        )
        INSERT INTO gold.DimOutcome (outcome_key, outcome, outcome_code, outcome_name, outcome_display_name, has_outcome_information)
        SELECT @MaxKey + CAST(ROW_NUMBER() OVER (ORDER BY outcome, outcome_code, outcome_name) AS int),
            outcome, outcome_code, outcome_name, COALESCE(outcome_name, outcome, outcome_code, 'Not recorded'), CAST(CASE WHEN outcome IS NOT NULL OR outcome_code IS NOT NULL OR outcome_name IS NOT NULL THEN 1 ELSE 0 END AS bit)
        FROM NewCombinations;
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## gold.usp_LoadFactStopSearch

```sql
CREATE OR ALTER PROCEDURE gold.usp_LoadFactStopSearch
    @ForceId varchar(100) = NULL,
    @SourceMonth varchar(7) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @ExpectedRows bigint, @LoadedRows bigint;
    DECLARE @LoadedAt datetime2(6) = SYSUTCDATETIME();
    SET @ForceId = NULLIF(TRIM(@ForceId), '');
    SET @SourceMonth = NULLIF(TRIM(@SourceMonth), '');
    BEGIN TRY
        BEGIN TRANSACTION;
        SELECT @ExpectedRows = COUNT_BIG(*) FROM silver.StopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        IF @ExpectedRows = 0 THROW 50020, 'No Silver records for selected scope.', 1;
        IF EXISTS (
            SELECT source_file, source_record_index FROM silver.StopSearch
            WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth)
            GROUP BY source_file, source_record_index HAVING COUNT_BIG(*) > 1
        ) THROW 50021, 'Duplicate source-file positions in Silver.', 1;
        DELETE FROM gold.FactStopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        INSERT INTO gold.FactStopSearch
        (date_key, force_key, demographic_key, search_profile_key, outcome_key, force_id, source_month, source_file, source_record_index, search_datetime_utc, search_datetime_uk, involved_person, outcome_linked_to_search, latitude, longitude, street_id, street_name, operation, operation_name, removal_of_outer_clothing, has_outcome_information, has_valid_coordinates, has_valid_timestamp, publication_month_matches_uk, search_count, gold_loaded_at_utc)
        SELECT dt.date_key, f.force_key, dm.demographic_key, sp.search_profile_key, o.outcome_key, s.force_id, s.source_month, s.source_file, s.source_record_index, s.search_datetime_utc, s.search_datetime_uk, s.involved_person, s.outcome_linked_to_search, s.latitude, s.longitude, s.street_id, s.street_name, s.operation, s.operation_name, s.removal_of_outer_clothing, s.has_outcome_information, s.has_valid_coordinates, s.has_valid_timestamp, s.publication_month_matches_uk, 1, @LoadedAt
        FROM silver.StopSearch s
        LEFT JOIN gold.DimDate dt ON dt.date_key = CASE WHEN s.search_date_uk IS NULL THEN 0
            ELSE CONVERT(int, CONVERT(varchar(8), s.search_date_uk, 112)) END
        LEFT JOIN gold.DimForce f ON f.force_id IS NOT DISTINCT FROM s.force_id
        LEFT JOIN gold.DimDemographic dm ON dm.age_range IS NOT DISTINCT FROM s.age_range
            AND dm.gender IS NOT DISTINCT FROM s.gender
            AND dm.self_defined_ethnicity IS NOT DISTINCT FROM s.self_defined_ethnicity
            AND dm.officer_defined_ethnicity IS NOT DISTINCT FROM s.officer_defined_ethnicity
LEFT JOIN gold.DimSearchProfile sp ON sp.search_type IS NOT DISTINCT FROM s.search_type
            AND sp.legislation IS NOT DISTINCT FROM s.legislation
            AND sp.object_of_search IS NOT DISTINCT FROM s.object_of_search
LEFT JOIN gold.DimOutcome o ON o.outcome IS NOT DISTINCT FROM s.outcome
            AND o.outcome_code IS NOT DISTINCT FROM s.outcome_code
            AND o.outcome_name IS NOT DISTINCT FROM s.outcome_name
        WHERE (@ForceId IS NULL OR s.force_id = @ForceId)
          AND (@SourceMonth IS NULL OR s.source_month = @SourceMonth);
        SELECT @LoadedRows = COUNT_BIG(*) FROM gold.FactStopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        IF @LoadedRows <> @ExpectedRows THROW 50022, 'Silver/Gold count mismatch.', 1;
        COMMIT TRANSACTION;
        SELECT 'Succeeded' AS load_status, @ExpectedRows AS silver_rows,
            @LoadedRows AS gold_rows, @LoadedRows - @ExpectedRows AS row_difference;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH;
END;
```

## gold.usp_RunFactStopSearchLoad

```sql
CREATE OR ALTER PROCEDURE gold.usp_RunFactStopSearchLoad
    @ForceId varchar(100) = NULL,
    @SourceMonth varchar(7) = NULL,
    @PipelineRunId varchar(100) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    IF @@TRANCOUNT > 0 THROW 50030, 'Run without an existing transaction.', 1;
    DECLARE @LoadId varchar(36) = CONVERT(varchar(36), NEWID());
    DECLARE @SourceRows bigint, @TargetRows bigint, @ErrorMessage varchar(8000);
    SET @ForceId = NULLIF(TRIM(@ForceId), '');
    SET @SourceMonth = NULLIF(TRIM(@SourceMonth), '');
    INSERT INTO audit.StopSearchGoldLoad
        (load_id, pipeline_run_id, force_id, source_month, started_at_utc, load_status)
    VALUES (@LoadId, @PipelineRunId, @ForceId, @SourceMonth, SYSUTCDATETIME(), 'Running');
    BEGIN TRY
        EXEC gold.usp_LoadFactStopSearch @ForceId = @ForceId, @SourceMonth = @SourceMonth;
        SELECT @SourceRows = COUNT_BIG(*) FROM silver.StopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        SELECT @TargetRows = COUNT_BIG(*) FROM gold.FactStopSearch WHERE (@ForceId IS NULL OR force_id = @ForceId)
          AND (@SourceMonth IS NULL OR source_month = @SourceMonth);
        UPDATE audit.StopSearchGoldLoad
        SET completed_at_utc = SYSUTCDATETIME(), silver_rows = @SourceRows,
            gold_rows = @TargetRows, load_status = 'Succeeded'
        WHERE load_id = @LoadId;
    END TRY
    BEGIN CATCH
        SET @ErrorMessage = CONCAT('Error ', ERROR_NUMBER(), ': ', ERROR_MESSAGE());
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        UPDATE audit.StopSearchGoldLoad
        SET completed_at_utc = SYSUTCDATETIME(), load_status = 'Failed', error_message = @ErrorMessage
        WHERE load_id = @LoadId;
        THROW;
    END CATCH;
END;
```

## Suggested non-production verification

Compare table metadata against the dictionary; compile each definition separately; run dimensions in order; execute a selected scope through the wrappers; verify source/target counts and audit fields; rerun and confirm stable dimension keys and unchanged totals. These are adoption checks, not tests executed against Fabric by this repository build. Use the documented corrected Gold pipeline scope expressions when checking parent orchestration.
