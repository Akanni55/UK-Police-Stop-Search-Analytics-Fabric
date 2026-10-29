# Evidence and export checklist

[Back to project](../README.md)

## Received and reviewed

- Processing-parent successful execution screenshot.
- Gold four-activity successful execution screenshot.
- Historical ingestion recovery screenshot.
- Semantic-model relationship screenshot.
- Silver audit screenshot (14,899 scoped records).
- Gold audit screenshot (1,436,597 full-scope records).
- API extraction child canvas and parameter screenshot.
- Reference notebook export: code reproduced inline in the [verified walkthrough](../notebooks/reference-walkthrough.md).
- Stop and Search notebook export: currently profiling code, reviewed [here](../notebooks/stop-search-profile-review.md).

Screenshot publication was explicitly approved by the owner and is complete; see the [evidence gallery](evidence-gallery.md).

## Still requested

1. Actual raw-to-Bronze writer code; the supplied Stop and Search notebook does not contain it.
2. Reference pipeline canvas and exported pipeline definitions.
3. Current Warehouse table DDL and stored-procedure definitions.
4. Three-layer reconciliation result capture, if available.
5. A scoped Gold audit after confirming parent parameter propagation.
6. Final report screenshots and the published report URL when ready.

Exports must be sanitised for credentials and unnecessary tenant-specific connection identifiers. Notebook outputs and Fabric attachment metadata are not included in the inline walkthroughs.

## Update — 29 October 2026

Received all six pipeline ZIP exports, the three-layer count screenshot (1,436,597 each), and a seven-procedure catalog screenshot. Pipeline definitions are now sanitised and displayed inline. Full SQL text is still required as a CSV/text export because the screenshot truncates the definition cells. Actual Bronze writer, final report and screenshot-publication approval remain outstanding. A reference-pipeline screenshot is now optional illustration rather than a configuration blocker because the export verifies its activities.

## Documentation completion decision

The owner has supplied the files available and requested completion without further file hunting. Seven full SQL procedure reconstructions and a complete Bronze writer reconstruction are now published inline with explicit provenance. A procedure CSV or original writer export would be optional future verification, not an outstanding blocker for this case-study documentation. Their reconstructed status must not be removed without verification. The remaining delivery item is the report when the owner finishes it; public screenshot publication has been approved and completed.

## Screenshot publication completed

The owner explicitly approved public publication on 29 October 2026. All nine supplied screenshots are now published and captioned in the [evidence gallery](evidence-gallery.md). Earlier references to pending screenshot approval are historical and resolved. The Power BI report remains in progress; code reconstructions remain labelled as reconstructions.
