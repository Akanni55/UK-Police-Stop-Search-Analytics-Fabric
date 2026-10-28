# Bronze reference notebook — verified export walkthrough

[Back to project](../README.md)

Reviewed from the owner-supplied `NB_Police_Load_Bronze_Reference.ipynb`. The code is displayed inline below; notebook output and Fabric attachment metadata are not published. These are Spark SQL cells, not Warehouse T-SQL.

The notebook creates Lakehouse schemas, reads retained reference JSON, replaces the reference Delta tables and rebuilds the request plan. Its `CREATE OR REPLACE TABLE` behaviour is a full reference refresh, not append-only history. The Lakehouse `silver` and `gold` schema creation cells are historical setup; the implemented analytical Silver/Gold tables live in the Warehouse.

## 1. Create Lakehouse schemas

```sql
CREATE SCHEMA IF NOT EXISTS bronze;
CREATE SCHEMA IF NOT EXISTS silver;
CREATE SCHEMA IF NOT EXISTS gold;
CREATE SCHEMA IF NOT EXISTS control;
CREATE SCHEMA IF NOT EXISTS audit;
```

## 2. Read the forces JSON

```sql
CREATE OR REPLACE TEMP VIEW raw_forces
USING json
OPTIONS (
    path 'Files/raw/police/forces/forces.json',
    multiLine 'true'
);
```

## 3. Preview forces

```sql
SELECT *
FROM raw_forces;
```

## 4. Replace the forces reference

```sql
CREATE OR REPLACE TABLE bronze.Forces
USING DELTA
AS
SELECT
    id,
    name,
    current_timestamp() AS ingested_at,
    'Files/raw/police/forces/forces.json' AS source_file
FROM raw_forces;
```

## 5. Read availability JSON

```sql
CREATE OR REPLACE TEMP VIEW raw_available_months
USING json
OPTIONS (
    path 'Files/raw/police/available_months/available_months.json',
    multiLine 'true'
);
```

## 6. Replace availability metadata

```sql
CREATE OR REPLACE TABLE bronze.AvailableMonths
USING DELTA
AS
SELECT
    date,
    `stop-and-search` AS stop_and_search_forces,
    current_timestamp() AS ingested_at,
    'Files/raw/police/available_months/available_months.json'
        AS source_file
FROM raw_available_months;
```

## 7. Read exploratory crime categories

Crime-category cells belong to the earlier exploration and are not required by the Stop and Search model. They still execute if this unchanged notebook is run end to end; remove or isolate them before claiming a Stop-and-Search-only reference refresh.

```sql
CREATE OR REPLACE TEMP VIEW raw_crime_categories
USING json
OPTIONS (
    path 'Files/raw/police/crime_categories/month=*/crime_categories.json',
    multiLine 'true'
);
```

## 8. Replace exploratory crime categories

```sql
CREATE OR REPLACE TABLE bronze.CrimeCategories
USING DELTA
AS
SELECT
    url,
    name,
    regexp_extract(
        input_file_name(),
        '/month=([0-9]{4}-[0-9]{2})/',
        1
    ) AS source_month,
    current_timestamp() AS ingested_at,
    input_file_name() AS source_file
FROM raw_crime_categories;
```

## 9. Build the availability-driven request plan

```sql
CREATE OR REPLACE TABLE control.StopSearchLoadPlan
USING DELTA
AS
SELECT DISTINCT
    m.date AS source_month,
    force_id
FROM bronze.AvailableMonths m
LATERAL VIEW explode(m.stop_and_search_forces) f AS force_id
WHERE m.date IS NOT NULL
  AND force_id IS NOT NULL
  AND trim(force_id) <> '';
```

## 10. Check coverage

```sql
SELECT
    COUNT(*) AS total_api_requests,
    COUNT(DISTINCT source_month) AS months_covered,
    COUNT(DISTINCT force_id) AS forces_covered,
    MIN(source_month) AS earliest_month,
    MAX(source_month) AS latest_month
FROM control.StopSearchLoadPlan;
```

## 11. Preview planned requests

```sql
SELECT *
FROM control.StopSearchLoadPlan
ORDER BY source_month DESC, force_id
LIMIT 20;
```

