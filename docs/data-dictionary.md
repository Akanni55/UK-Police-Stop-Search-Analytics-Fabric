# Data dictionary and metric contract

[Back to project](../README.md)

## Bronze and metadata

| Table | Grain / important fields |
|---|---|
| `bronze.stopsearch` | One parsed array record: `source_file`, `source_record_index` (zero-based), `raw_payload`, `force_id`, `source_month`, `ingested_at` |
| `bronze.forces` | Reference force ID/name with ingestion lineage |
| `bronze.availablemonths` | Publication month and array of contributing Stop and Search force IDs, with ingestion metadata |
| `control.stopsearchloadplan` | Distinct planned force/publication-month pair |
| `control.stopsearchretryplan` | Reduced set of requests selected for recovery |
| `audit.stopsearchfilecheck` | File validation status, record count, source metadata, error and check time |

Lowercase SQL endpoint identifiers should be confirmed against the environment when reproducing. The exact production-use source references in the developed SQL are `[LH_Uk_Police_Crime].[bronze].[stopsearch]` and `[LH_Uk_Police_Crime].[bronze].[forces]`.

## Silver StopSearch

| Columns | Stored type | Meaning |
|---|---|---|
| `force_id` | varchar(100), required | API request force identifier |
| `source_month` | varchar(7), required | Requested publication month, YYYY-MM |
| `source_file` | varchar(8000), required | Raw source path; workspace-specific paths are not published as examples |
| `source_record_index` | int, required | Array position within a file version |
| `source_datetime` | varchar(100) | Original extracted datetime text |
| `search_datetime_utc`, `search_datetime_uk` | datetime2(6) | Converted UTC and UK local wall-clock timestamps |
| `search_date_uk` | date | UK local event date |
| `search_type` | varchar(4000) | Published person/vehicle search classification |
| `involved_person` | bit, nullable | Published person involvement flag |
| `legislation`, `object_of_search` | varchar(4000) | Published legal power and search object |
| `age_range`, `gender` | varchar(4000) | Published demographic categories |
| `self_defined_ethnicity`, `officer_defined_ethnicity` | varchar(4000) | Separate published ethnicity classifications; not merged |
| `outcome` | varchar(4000) | Normalised top-level outcome; current false/NULL caveat applies |
| `outcome_code`, `outcome_name` | varchar(4000) | Nested outcome object values |
| `outcome_linked_to_search` | bit, nullable | Whether outcome links to object of search |
| `latitude`, `longitude` | decimal(12,8), nullable | Parsed approximate coordinates |
| `street_id` | bigint, nullable | Published location identifier; not a search identifier |
| `street_name` | varchar(4000) | Approximate street/location label |
| `operation` | bit, nullable | Published operation flag |
| `operation_name` | varchar(4000) | Published operation name |
| `removal_of_outer_clothing` | bit, nullable | Mapped from source `removal_of_more_than_outer_clothing` |
| `has_outcome_information` | bit, required | Any retained outcome text/code/name present |
| `has_valid_coordinates` | bit, required | Both coordinates numeric and within global coordinate bounds |
| `has_valid_timestamp` | bit, required | Source datetime parsed successfully |
| `publication_month_matches_uk` | bit, required | Publication month equals UK event YYYY-MM |
| `silver_loaded_at` | datetime2(6), required | SQL load timestamp recorded in UTC |

Silver keeps all source records. NULL boolean values remain unknown, rather than being defaulted to false. Offset-bearing timestamps are parsed as datetimeoffset expressions and stored as datetime2 values after timezone conversion; retain the UTC column for ordering across daylight-saving clock changes.

## Gold dimensions

| Table | Key | Attributes |
|---|---|---|
| `DimDate` | `date_key` int | `calendar_date` date; year, quarter, month number, month name, year-month, day number, weekday number/name, weekend flag |
| `DimForce` | `force_key` int | `force_id`, `force_name` |
| `DimDemographic` | `demographic_key` int | Age range, gender and both ethnicity classifications |
| `DimSearchProfile` | `search_profile_key` int | Search type, legislation, object of search |
| `DimOutcome` | `outcome_key` int | Outcome, nested code/name, display label, information-present flag |

All dimension keys are required. Descriptive attributes can be NULL. Date uses deterministic calendar keys. Other keys are serially allocated and preserved on later loads. Key 0 represents an all-missing combination, not every partially missing record.

`DimDate` columns: `calendar_year`, `calendar_quarter`, `month_number`, `day_of_month`, `day_of_week_number` are int; `month_name` and `day_name` varchar(20); `year_month` varchar(7); `is_weekend` bit. `DimForce.force_id` is varchar(100); its name and all demographic/search-profile/outcome text attributes use varchar(4000). `DimOutcome.has_outcome_information` is a required bit.

## Gold FactStopSearch

| Group | Columns / contract |
|---|---|
| Required dimension keys | `date_key`, `force_key`, `demographic_key`, `search_profile_key`, `outcome_key` — int |
| Required lineage | `force_id`, `source_month`, `source_file`, `source_record_index` — same types as Silver |
| Event time | `search_datetime_utc`, `search_datetime_uk` — datetime2(6), nullable |
| Search flags | `involved_person`, `outcome_linked_to_search`, `operation`, `removal_of_outer_clothing` — nullable bit |
| Location | `latitude`, `longitude`, `street_id`, `street_name` — same types as Silver |
| Operation label | `operation_name` — varchar(4000), nullable |
| Quality flags | The four Silver quality flags — required bit |
| Additive measure | `search_count` — required int, populated with 1 |
| Load timestamp | `gold_loaded_at_utc` — required datetime2(6) |

No enforced primary/foreign keys were created in the supplied fact/dimension DDL. Procedural validation and model relationship settings provide checks, not database-enforced uniqueness guarantees.

## Audit contracts

Both SQL audit tables contain `load_id` varchar(36), `pipeline_run_id` varchar(100), `force_id` varchar(100), `source_month` varchar(7), UTC start/completion datetime2(6), source/target bigint counts, `load_status` varchar(20), `error_message` varchar(8000). Silver uses `bronze_rows`/`silver_rows`; Gold uses `silver_rows`/`gold_rows`. Force/month NULL means the full corresponding scope. Counts can remain NULL on failure.
