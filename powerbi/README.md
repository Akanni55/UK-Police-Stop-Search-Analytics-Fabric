# UK Police Stop and Search Insights

[Back to project](../README.md)

## Publication status

**Report authoring in progress — public deployment pending.**

| Artefact | Agreed name |
|---|---|
| Report display name | UK Police Stop and Search Insights |
| Power BI file | `UK_Police_Stop_Search_Insights.pbix` |
| Semantic model | `SM_Police_StopSearch` |
| Public report URL | Not yet supplied |
| Report screenshot | Awaiting completed report |

The model and starter measures were prepared during implementation. A live report is not represented as deployed. Once available, add the actual URL here and in the main README; a report name is not a routable Power BI URL.

## Model relationships

The model is created from six physical Gold Warehouse tables in Direct Lake mode. Each relationship is active, single-direction and one-to-many from dimension to fact.

| One side | Many side |
|---|---|
| `DimDate[date_key]` | `FactStopSearch[date_key]` |
| `DimForce[force_key]` | `FactStopSearch[force_key]` |
| `DimDemographic[demographic_key]` | `FactStopSearch[demographic_key]` |
| `DimSearchProfile[search_profile_key]` | `FactStopSearch[search_profile_key]` |
| `DimOutcome[outcome_key]` | `FactStopSearch[outcome_key]` |

The first unfiltered Total Searches card should show **1,436,597** for this snapshot. Slicing `DimDate` uses the UK event date; `source_month` is the API publication month. Label visuals accordingly.

## Measures, with definitions

### Total Searches

One unit per source record, not a distinct-person count.

```dax
Total Searches =
SUM ( FactStopSearch[search_count] )
```

### Searches with Recorded Outcome

Counts records retaining outcome information after the current Silver normalisation. `KEEPFILTERS` intersects this condition with existing report filters.

```dax
Searches with Recorded Outcome =
CALCULATE (
    [Total Searches],
    KEEPFILTERS (
        FactStopSearch[has_outcome_information] = TRUE ()
    )
)
```

### Outcome Recording Rate

A data-presence measure under current transformation rules, not an arrest rate or effectiveness measure. The source's literal false outcome is currently normalised to NULL; this limitation must accompany interpretation until corrected.

```dax
Outcome Recording Rate =
DIVIDE (
    [Searches with Recorded Outcome],
    [Total Searches]
)
```

### Searches with Valid Coordinates

A numeric range check, not exact-location accuracy or force-boundary validation.

```dax
Searches with Valid Coordinates =
CALCULATE (
    [Total Searches],
    KEEPFILTERS (
        FactStopSearch[has_valid_coordinates] = TRUE ()
    )
)
```

### Coordinate Coverage Rate

```dax
Coordinate Coverage Rate =
DIVIDE (
    [Searches with Valid Coordinates],
    [Total Searches]
)
```

Use whole-number formatting with thousands separators for counts and one-decimal percentage formatting for rates. `DIVIDE` returns BLANK for an empty denominator by default.

## Proposed report pages

These are report design suggestions, not screenshots of a completed product.

1. **Overview:** total searches, monthly trend, force volumes and selected period.
2. **Search patterns:** search type, legislation and object of search.
3. **Demographics and recorded outcomes:** published age, gender, ethnicity and outcome breakdowns, with interpretation notes.
4. **Coverage and data quality:** outcome/coordinate coverage and force/month availability context.

Do not show an unavailable force/month as a proven zero. For demographic comparisons, display counts as counts; population-adjusted rates require a suitable denominator dataset. Mapping must label coordinates as approximate.

## Before publication

Confirm measure totals against Gold, inspect unknown members, verify slicer direction and distinguish source-month from event-date trends. Add month/day sorting where needed. Hide surrogate keys and technical lineage columns from normal report authoring while retaining them in the model for diagnosis. Model refresh/reframing after warehouse updates must be configured and validated; the processing parent does not yet include a semantic-model refresh activity.

The NULL-date unknown member means the current DimDate calendar column is not ready for a conventional date-table designation without adjustment. Resolve that design before introducing classic year-on-year DAX. RLS and production deployment promotion have not been implemented or claimed.
