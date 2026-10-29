# Implementation status, limitations and next steps

[Back to project](../README.md)

## What is implemented

Historical availability-driven API extraction; retained force/month JSON; Bronze source-payload lineage; SQL-based Silver parsing; UK/UTC timestamps; quality flags; five dimensions; a source-record fact; transactional scope replacement; count reconciliation; selected-scope reruns; ordered Silver/Gold parent processing; SQL audit wrappers; semantic-model relationships and starter DAX.

## Important analytical limitations

**Coverage:** a 42-force snapshot is not every UK force or every possible search. Missing force/month availability is not zero activity. Scotland-wide coverage is not established. The UK label identifies the source context rather than a completeness guarantee.

**Identity:** no stable event or person identifier is available in the ingested payload contract. File path plus array index is lineage within a file version. It cannot track individuals or establish whether separate records refer to the same person.

**Outcomes:** the current Silver mapping converts literal false to NULL. The API documentation gives false a meaning distinct from an unprovided value. Preserve that distinction in a future outcome state field, profile the affected payloads and reprocess before treating the current recording-rate measure as an exact missingness measure. Original payloads are retained in Bronze.

**Time:** publication month differs from local event month. UK daylight-saving transitions are handled through timezone conversion; some source forces provide dates with no meaningful time-of-day, so midnight patterns cannot automatically be interpreted as operational patterns.

**Locations:** published coordinates are approximate. Coordinate validity checks are numeric only. Map coverage excludes missing coordinates and must not imply exact incident addresses.

**Demographics:** preserve self-defined and officer-defined ethnicity separately. Count comparisons do not establish population-adjusted disproportionality or causal effects.

## Operational limitations

- Source-to-Bronze ingestion is separate from the tested Bronze-to-Gold processing parent.
- No automatic new-month discovery-to-report schedule or ingestion success watermark has been demonstrated.
- Current raw paths do not retain immutable versions for every correction.
- Two notebook exports and seven screenshots have been reviewed. Public screenshot publication has been approved and completed; six pipeline exports are now verified; full procedure text and the actual raw-to-Bronze writer remain pending.
- Exports confirm the Gold fact activity sends empty scope literals despite correct parent mappings. The documented expression correction still needs applying and a scoped rerun.
- The supplied Stop and Search notebook is a profiling notebook, not a writer, and its final profiling query uses the wrong FROM source for its CTE aliases. Actual Bronze replay write behaviour remains unverified.
- Manual dimension key allocation requires serial execution; the database DDL does not enforce key uniqueness.
- Audit wrappers cover Silver and Gold fact loads; dimension stages are monitored in pipeline history.
- Cancellation can leave Running audit rows. Data commits and audit completion are not a distributed exactly-once transaction.
- Empty-source scopes are rejected; legitimate source deletions to an empty response need an explicit authorised reconciliation policy.
- No production RLS, CI/CD environment promotion, alerting, quarantine process or automated schema-evolution contract is claimed.
- No benchmarked speedup or capacity-cost reduction is claimed.
- Report publication and model refresh integration are pending.

## Prioritised next work

1. Verify sanitised notebook, pipeline and SQL exports against this documentation and add execution screenshots.
2. Preserve false vs missing outcome states and revalidate dependent KPIs.
3. Publish the report and replace the report-status link with its real URL.
4. Add immutable ingestion versions/checksums, a success ledger and scoped Bronze replay.
5. Implement no-overlap enforcement, shared parent-run correlation and automated integrity checks.
6. Add monthly availability refresh and reviewed historical-correction windows.
7. Benchmark controlled ingestion concurrency, scoped vs full loads and representative star-schema queries.
8. Implement environment-specific deployment, access controls and operational alerts as separate tested changes.

This roadmap adapts the original broader crime-intelligence assignment to the delivered Stop and Search project. Unimplemented assignment requirements are not presented as achievements.

## Reconstructed implementation coverage

Complete SQL and Bronze writer code is now available inline. These are reconstructed implementations, not verified live exports; they have not been deployed or executed in Fabric by this documentation update. Local checks cover Python syntax, JSON parsing and column-contract shape only. The Bronze reconstruction replaces the complete target snapshot and requires isolation/backup validation before use. The Gold scope mapping correction remains documented but unconfirmed in Fabric. No additional owner artefacts are required to complete this documentation pass; future exports can strengthen provenance.
