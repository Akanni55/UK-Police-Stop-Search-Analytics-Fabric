# Stop and Search profiling notebook — verified export review

[Back to project](../README.md)

The supplied file is named `NB_Police_Load_Bronze_StopSearch.ipynb`, but its current contents read existing Bronze data, create a temporary profiling view and run checks. **It does not contain raw-file parsing or a Bronze table write.** The actual writer used for the completed historical load is still requested.

The export is recorded faithfully below as Spark SQL. It is not represented as a deployable ingestion notebook, and it has not been executed during this repository review.

## Exported cell 1

```sql
CREATE OR REPLACE TEMP VIEW stop_search_profile AS
SELECT
    force_id,
    source_month,
    source_file,
    source_record_index,

    get_json_object(
        raw_payload, '$.datetime'
    ) AS search_datetime,

    get_json_object(
        raw_payload, '$.type'
    ) AS search_type,

    get_json_object(
        raw_payload, '$.involved_person'
    ) AS involved_person,

    get_json_object(
        raw_payload, '$.age_range'
    ) AS age_range,

    get_json_object(
        raw_payload, '$.gender'
    ) AS gender,

    get_json_object(
        raw_payload, '$.self_defined_ethnicity'
    ) AS self_defined_ethnicity,

    get_json_object(
        raw_payload, '$.officer_defined_ethnicity'
    ) AS officer_defined_ethnicity,

    get_json_object(
        raw_payload, '$.legislation'
    ) AS legislation,

    get_json_object(
        raw_payload, '$.object_of_search'
    ) AS object_of_search,

    get_json_object(
        raw_payload, '$.outcome'
    ) AS outcome,

    get_json_object(
        raw_payload, '$.outcome_object.id'
    ) AS outcome_code,

    get_json_object(
        raw_payload, '$.outcome_object.name'
    ) AS outcome_name,

    get_json_object(
        raw_payload, '$.outcome_linked_to_object_of_search'
    ) AS outcome_linked_to_object_of_search,

    get_json_object(
        raw_payload, '$.location.latitude'
    ) AS latitude,

    get_json_object(
        raw_payload, '$.location.longitude'
    ) AS longitude,

    get_json_object(
        raw_payload, '$.location.street.id'
    ) AS street_id,

    get_json_object(
        raw_payload, '$.location.street.name'
    ) AS street_name,

    get_json_object(
        raw_payload, '$.operation'
    ) AS operation,

    get_json_object(
        raw_payload, '$.operation_name'
    ) AS operation_name,

    get_json_object(
        raw_payload, '$.removal_of_more_than_outer_clothing'
    ) AS removal_of_more_than_outer_clothing

FROM bronze.StopSearch;
```

## Exported cell 2

```sql
SELECT *
FROM bronze.StopSearch
LIMIT 5;
```

## Exported cell 3

```sql
WITH prepared AS (
    SELECT
        *,
        TRY_CAST(
            NULLIF(trim(search_datetime), '')
            AS TIMESTAMP
        ) AS parsed_datetime,

        TRY_CAST(
            NULLIF(trim(latitude), '')
            AS DOUBLE
        ) AS parsed_latitude,

        TRY_CAST(
            NULLIF(trim(longitude), '')
            AS DOUBLE
        ) AS parsed_longitude

    FROM stop_search_profile
)
SELECT
    COUNT(*) AS total_records,

    SUM(CASE
        WHEN search_datetime IS NULL
          OR trim(search_datetime) = ''
        THEN 1 ELSE 0
    END) AS missing_datetime,

    SUM(CASE
        WHEN search_datetime IS NOT NULL
         AND trim(search_datetime) <> ''
         AND parsed_datetime IS NULL
        THEN 1 ELSE 0
    END) AS invalid_datetime,

    SUM(CASE
        WHEN search_type IS NULL
          OR trim(search_type) = ''
        THEN 1 ELSE 0
    END) AS missing_search_type,

    SUM(CASE
        WHEN outcome IS NULL
          OR trim(outcome) = ''
        THEN 1 ELSE 0
    END) AS missing_outcome,

    SUM(CASE
        WHEN latitude IS NULL
          OR longitude IS NULL
          OR trim(latitude) = ''
          OR trim(longitude) = ''
        THEN 1 ELSE 0
    END) AS missing_coordinates,

    SUM(CASE
        WHEN (
            latitude IS NOT NULL
            AND trim(latitude) <> ''
            AND (
                parsed_latitude IS NULL
                OR parsed_latitude NOT BETWEEN -90 AND 90
            )
        )
        OR (
            longitude IS NOT NULL
            AND trim(longitude) <> ''
            AND (
                parsed_longitude IS NULL
                OR parsed_longitude NOT BETWEEN -180 AND 180
            )
        )
        THEN 1 ELSE 0
    END) AS invalid_coordinates,

    MIN(parsed_datetime) AS earliest_search,
    MAX(parsed_datetime) AS latest_search

FROM bronze.StopSearch;
```

## Review finding: final query source

The final cell defines `prepared` from `stop_search_profile` but selects from `bronze.StopSearch`. Its projected aliases such as `parsed_datetime`, `parsed_latitude` and `parsed_longitude` are defined in the CTE, not in the documented Bronze contract. For the intended profiling query, the final FROM should refer to `prepared`:

```sql
-- Intended final source for exported cell 3:
FROM prepared;
```

This is a static code-review finding. The supplied export is preserved above without silently rewriting it as a previously successful execution. The deployed Warehouse transformation remains the implemented Silver path.
