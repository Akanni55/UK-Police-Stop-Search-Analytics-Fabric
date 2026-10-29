# API ingestion and Bronze walkthrough

[Back to project](../README.md)

## Start with metadata, not a hardcoded calendar

The API base is `https://data.police.uk/api/`. The project uses:

| Relative endpoint | Purpose |
|---|---|
| `forces` | Retrieve reference identifiers and names |
| `crimes-street-dates` | Retrieve available months and each month's `stop-and-search` force list |
| `stops-force?force={id}&date={YYYY-MM}` | Download one advertised force/month response |

The reference count is not the extraction count. The reference snapshot had 44 forces; 42 contributed loaded records. Expanding the monthly availability arrays produced 1,427 request pairs rather than the full Cartesian product of every force and month.

The following is an explanatory equivalent of the load-plan expansion, not an exported notebook cell:

```python
from pyspark.sql import functions as F

load_plan = (
    spark.table("bronze.AvailableMonths")
    .select(
        F.col("date").alias("source_month"),
        F.explode("stop_and_search_forces").alias("force_id")
    )
    .dropDuplicates(["source_month", "force_id"])
)
# Write using the schema/table identifier supported by your attached Lakehouse.
```

Deduplicating the request plan is appropriate; deduplicating search records by descriptive attributes is not.

## Reusable extraction child

`PL_Police_Load_Stop_Search` accepts `p_force` and `p_month`. Its REST connection points at the base API. The Copy activity constructs a relative request:

```text
@concat('stops-force?force=', pipeline().parameters.p_force,
        '&date=', pipeline().parameters.p_month)
```

For the selected month and force:

```text
https://data.police.uk/api/stops-force?force=metropolitan&date=2025-08
```

Raw output is partitioned by the request metadata:

```text
Files/raw/police/stop_search/force=metropolitan/month=2025-08/stop_search.json
```

A representative dynamic folder expression in the child is:

```text
@concat('raw/police/stop_search/force=', pipeline().parameters.p_force,
        '/month=', pipeline().parameters.p_month)
```

The destination file is JSON array content. No API pagination was configured for these responses. The exact exported activity is now available in the [sanitised inline pipeline walkthrough](verified-pipelines.md).

## Historical orchestration and recovery

`PL_Police_Load_All_Stop_Search` uses a Lookup of the request plan, a ForEach and an invoked extraction child with completion waiting enabled. The historical run used sequential requests. When four requests failed, a retry-plan table isolated the remaining work instead of restarting all downloads.

The parent was last left pointing at the retry plan. Review its Lookup before reusing it. Keep that table while the pipeline still references it. Do not treat this archived backfill pipeline as a configured monthly scheduler.

## Raw-to-Bronze boundary

The implemented Bronze table contains one row per array element, retaining the payload and its position. However, the newly supplied notebook named `NB_Police_Load_Bronze_StopSearch` contains profiling queries rather than the writer. See the [verified export review](../notebooks/stop-search-profile-review.md); the actual raw-to-Bronze writer remains requested. The raw files remain the authoritative response artefacts; parsing and JSON reserialization need not preserve original byte formatting.

During validation, a response contained a UTF-8 BOM before the opening array bracket. Strip only the leading BOM before JSON parsing. The project checked every planned file as valid array content; all loaded files were non-empty.

This fragment illustrates the lineage principle after parsing an array column named `records`:

```python
# Explanatory fragment; complete deployed notebook export is pending.
exploded = parsed.select(
    "source_file", "force_id", "source_month",
    F.posexplode("records").alias("source_record_index", "record")
)
bronze = exploded.select(
    "source_file", "source_record_index", "force_id", "source_month",
    F.to_json("record").alias("raw_payload"),
    F.current_timestamp().alias("ingested_at")
)
```

The file audit table records source path, force, month, validation status, record count, error and check time. The file record-count sum reconciled to 1,436,597 Bronze rows.

## Verified reference orchestration

The reference export confirms forces Copy → availability Copy → Lookup months → sequential monthly crime-category Copy → reference notebook. Thus the current refresh still includes exploratory category work even though the analytical model is Stop and Search only. Isolate those cells and activities before removing that dependency. The request recovery parent reads the retry table, not the full load plan.

## Replay and limitations

Raw retention allows transformation recovery without another API extraction. Before rerunning the Bronze notebook, inspect its write mode: the deployed export has not yet been attached here, so this documentation does not assert that its append/overwrite behaviour is incremental-safe. Scope-safe Silver and Gold replacement is implemented separately.

Source revisions are not versioned immutably by the current force/month path. Add extraction versions, checksums and a successful ingestion ledger before claiming historical revision tracking or fully incremental source ingestion. Consult the [official call-limit guidance](https://data.police.uk/docs/api-call-limits/) before changing concurrency; no throughput improvement has been benchmarked here.

## Complete writer reconstruction now available

A [complete reconstructed Bronze writer](../notebooks/bronze-writer-reconstruction.md) now implements the established lineage contract from retained JSON. It is not the recovered historical notebook: the supplied export lacks those cells. It performs a guarded full-snapshot overwrite and is not an incremental ingestion component. No existing Fabric data was changed. Validate in an isolated Lakehouse before adoption; no further owner file search is required for this documentation pass.
