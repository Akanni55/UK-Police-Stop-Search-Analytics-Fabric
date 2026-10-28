# Business case and implemented scope

[Back to project](../README.md)

## Scenario

A fictional public-sector analytics team needs a consistent view of published Stop and Search activity across contributing police forces. Manually assembling monthly API responses makes historical analysis difficult and obscures differences in data completeness. The engineering objective is to deliver a traceable, repeatable dataset suitable for reporting and for inspection by another engineer.

The case study is informed by a broader crime-intelligence assignment. The delivered scope was deliberately narrowed to Stop and Search. No crime fact, ASB fact or separate crime-outcome fact is implemented here.

## Objectives and delivered mechanisms

| Objective | Implementation |
|---|---|
| Avoid arbitrary regional restrictions | Load the force/month pairs advertised in the captured availability metadata |
| Retain historical source data | Store force/month JSON files in OneLake |
| Support reproducible transformations | Keep raw payload, source file, array position and ingestion metadata in Bronze |
| Standardise analysis fields | Parse nested JSON and convert types in Warehouse SQL |
| Handle incomplete data visibly | Preserve rows and expose quality flags rather than silently dropping incomplete records |
| Enable reliable aggregation | Load a source-record fact and five dimensions |
| Support repairs and reruns | Parameterised force/month replacement with transactions and reconciliation |
| Make operations inspectable | Persist Silver and Gold fact audit records linked to pipeline runs |
| Deliver an analytical product | Semantic model prepared; report authoring and publication remain in progress |

## Analytical questions

1. How do published search volumes vary over UK event dates and publication months?
2. How does recorded activity differ across contributing forces?
3. What search types, legislation and objects of search are represented?
4. How do volumes and recorded outcomes vary by published demographic categories?
5. How complete are outcome and coordinate fields by force and period?
6. Which reporting periods require investigation because of coverage gaps or publication/event-date mismatches?

These are descriptive questions. Raw volume differences do not establish comparative population risk, discriminatory practice or operational effectiveness.

## Scope contract

- Dataset: force-level Stop and Search responses and supporting reference metadata.
- Snapshot: 36 publication months from September 2023 through August 2026.
- Contributing data: 42 forces across 1,427 available force/month combinations.
- Scope is availability-driven. An unavailable combination is not interpreted as zero searches.
- No arbitrary Newcastle-only restriction remains in the implemented scope.
- Source updates can change availability, values and file ordering. Snapshot-local record lineage is not a permanent incident ID.
- The optional crime-category reference acquired during exploration is not required by the final Stop and Search model.

## Acceptance evidence

The project confirmed 1,436,597 records in Bronze, Silver and Gold, zero reconciliation difference, a successful selected force/month rerun and successful ordered parent processing. See [validation](../tests/validation.md). No benchmark uplift, production SLA, security certification or fully automated monthly source ingestion is claimed.
