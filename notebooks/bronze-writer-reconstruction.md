# Raw JSON to Bronze — reconstructed writer

[Back to project](../README.md)

## Provenance and execution boundary

This is a **new reconstruction of the established Bronze contract**, not a recovered original notebook cell or a verified deployed export. The original writer is unavailable in the supplied notebook, which was later used for profiling. Historical ingestion and the existing Bronze count are separately evidenced. This reconstruction has been checked for Python syntax and reviewed against that contract; it has not been executed in Fabric.

It reads retained JSON only, removes a leading BOM, rejects non-array/error bodies, preserves one row per array object, retains zero-based position and serialises the object into `raw_payload`. No event-attribute deduplication is performed. Empty files are allowed as valid zero-record arrays, but an entirely empty snapshot is rejected.

**This is a full-snapshot overwrite, not an incremental writer.** Do not run it on a partial raw folder or run it against the working Lakehouse merely to produce documentation. Validate it in an isolated Lakehouse copy first. Missing expected force/month files cannot be detected by syntax checks alone; compare the raw inventory against the captured request plan before any production-target rebuild. It does not create a new immutable raw version or rewrite the existing file audit table.

## Complete reconstructed cell

```python
import json
from pyspark.sql import functions as F, types as T

# Attach LH_Uk_Police_Crime as the default Lakehouse.
# Explicit full-snapshot rebuild from retained files: no API calls.
raw_root = "Files/raw/police/stop_search"
target = "bronze.stopsearch"

record_schema = T.StructType([
    T.StructField("source_record_index", T.IntegerType(), False),
    T.StructField("raw_payload", T.StringType(), False)
])
result_schema = T.StructType([
    T.StructField("records", T.ArrayType(record_schema), True),
    T.StructField("error", T.StringType(), True)
])

def reject_constant(value):
    raise ValueError("Non-standard JSON constant: " + value)

@F.udf(result_schema)
def parse_response(text):
    try:
        value = json.loads(text.lstrip("\ufeff"), parse_constant=reject_constant)
        if not isinstance(value, list):
            raise ValueError("Expected a JSON array")
        if any(not isinstance(record, dict) for record in value):
            raise ValueError("Every array element must be an object")
        records = [
            (index, json.dumps(record, ensure_ascii=False,
                               separators=(",", ":"), allow_nan=False))
            for index, record in enumerate(value)
        ]
        return records, None
    except Exception as exc:
        return None, str(exc)[:2000]

raw = (
    spark.read.option("wholetext", True)
    .option("recursiveFileLookup", True)
    .option("pathGlobFilter", "stop_search.json")
    .text(raw_root)
    .withColumn("source_file", F.input_file_name())
    .withColumn("force_id", F.regexp_extract(
        "source_file", r"/force=([^/]+)/", 1))
    .withColumn("source_month", F.regexp_extract(
        "source_file", r"/month=([0-9]{4}-[0-9]{2})/", 1))
)
parsed = raw.withColumn("parsed", parse_response("value")).cache()
try:
    file_count = parsed.count()
    if file_count == 0:
        raise ValueError("No raw files found; target unchanged")

    invalid = parsed.filter(
        F.col("parsed.error").isNotNull()
        | (F.col("force_id") == "")
        | (F.col("source_month") == "")
        | F.to_date(F.concat(F.col("source_month"), F.lit("-01"))).isNull()
    )
    if invalid.limit(1).count():
        invalid.select("source_file", "parsed.error").show(20, truncate=False)
        raise ValueError("Invalid raw input; target unchanged")

    expected = parsed.select(
        F.sum(F.size("parsed.records")).alias("n")
    ).first()["n"]
    if not expected:
        raise ValueError("Snapshot has no records; target unchanged")

    bronze = (
        parsed.select("source_file", "force_id", "source_month",
                      F.explode("parsed.records").alias("record"))
        .select("source_file", "force_id", "source_month",
                F.col("record.source_record_index").alias("source_record_index"),
                F.col("record.raw_payload").alias("raw_payload"),
                F.current_timestamp().alias("ingested_at"))
    )
    # An overwrite replaces the complete target snapshot, not one force/month.
    # Only run against a complete, reviewed raw snapshot with no concurrent writes.
    bronze.write.format("delta").mode("overwrite").saveAsTable(target)
    actual = spark.table(target).count()
    if actual != expected:
        raise RuntimeError(f"Post-write count mismatch: {expected} != {actual}")
    print({"files": file_count, "expected_records": expected,
           "bronze_records": actual, "difference": actual - expected})
finally:
    parsed.unpersist()
```

## Verification before adoption

1. Attach an isolated Lakehouse with schemas enabled and a copy of the intended raw snapshot.
2. Compare file paths/pairs against the captured load plan; a future API availability refresh is not a substitute for that historical plan.
3. Confirm the notebook's default Lakehouse and target identifier.
4. Run serially with no simultaneous raw updates. The writer holds a response array per file in worker memory and is not a streaming large-response parser.
5. For this historical snapshot, reconcile 1,427 files and 1,436,597 records; these counts are snapshot evidence, not hardcoded future requirements.
6. Inspect sample raw payloads, indices and both false and NULL outcome values.

Delta overwrite publishes the new table data as a transaction, but the post-write count check does not undo a committed write on failure. Keep a recovery copy/version before adoption. The existing working data is not modified by publishing this documentation.
