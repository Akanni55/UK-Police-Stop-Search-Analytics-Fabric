# Source register and attribution

[Back to project](../README.md)

## Public data

Source: **data.police.uk**, accessed through its public JSON API. Source documentation was checked on 28 October 2026. Project counts refer to the captured and loaded snapshot, not a live query executed by this documentation build.

| Resource | Link |
|---|---|
| API documentation | https://data.police.uk/docs/ |
| Force reference dictionary | https://data.police.uk/docs/method/forces/ |
| Availability dictionary | https://data.police.uk/docs/method/crimes-street-dates/ |
| Force-level Stop and Search dictionary | https://data.police.uk/docs/method/stops-force/ |
| Publication context and location handling | https://data.police.uk/about/ |
| API call limits | https://data.police.uk/docs/api-call-limits/ |
| Open Government Licence v3.0 | https://www.nationalarchives.gov.uk/doc/open-government-licence/version/3/ |

Contains public sector information licensed under the Open Government Licence v3.0. Source attribution does not imply endorsement of this independent project. This attribution concerns source data; the repository has not selected a separate blanket licence for original project code or documentation.

The source's force/month response dictionary supplies field meanings. The project's [data dictionary](data-dictionary.md) describes how those fields are retained and transformed. Pay particular attention to published approximate locations and the documented distinction between a false outcome and an omitted value.

## Platform references

- [Create a Fabric Warehouse semantic model](https://learn.microsoft.com/en-us/fabric/data-warehouse/create-semantic-model)
- [Stored procedure activity](https://learn.microsoft.com/en-us/fabric/data-factory/stored-procedure-activity)
- [Invoke pipeline activity](https://learn.microsoft.com/en-us/fabric/data-factory/invoke-pipeline-activity)
- [Cross-database Warehouse queries](https://learn.microsoft.com/en-us/fabric/data-warehouse/query-warehouse)
- [Warehouse transactions](https://learn.microsoft.com/en-us/fabric/data-warehouse/transactions)
- [Null-safe SQL comparison](https://learn.microsoft.com/en-us/sql/t-sql/queries/is-distinct-from-transact-sql)

Platform features and UI labels can change. Use the current official documentation when reproducing the project.
