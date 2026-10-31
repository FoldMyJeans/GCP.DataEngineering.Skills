# Reporting on BigQuery

Building the report is analyst work. Building the table and the pipeline that refreshes it is data engineering work. Most slow or wrong dashboards are fixed in the table, not the report.

## Choose the tool

| | Data Studio (used to be Looker Studio) | Looker |
| --- | --- | --- |
| Builder | Anyone, drag and drop | A data team writes a LookML model first |
| Metric definitions | Per report | Once, reused by every dashboard |
| Cost | Free (a paid Pro version adds team workspaces and support) | Licensed |
| Pick it when | A quick report or one team | One trusted definition across many teams |

They are different products. A BigQuery view is not the same as a governed metric: it gives one query, while LookML gives one definition that every dashboard reuses.

For spreadsheet users, Connected Sheets puts the Sheets interface on live BigQuery data with no export.

## Connect it

- The BigQuery connector can read a table in a shared project, not only your own.
- Check which account you are signed in as before connecting data. The avatar tells you. A leftover account in an incognito window broke one sign-in, and closing every incognito window and starting again fixed it.
- Searching "Data Studio" in the Cloud console opens the Data Studio Pro product page. Open the Data Studio site directly instead.
- Expect both the old and new product names on screen.

## Data credentials

Two separate settings, and mixing them up is the common mistake.

| Setting | Decides |
| --- | --- |
| Share | Who can open the report |
| Data credentials | Whose login runs the query |

With owner's credentials, every viewer sees the owner's data, including viewers with no access to the table. With viewer's credentials, each person needs their own access, and row-level rules apply to them.

Use viewer's credentials when each person should see only what they are allowed to. Not run here: viewer's credentials against a table with row-level security.

## Dimensions and metrics

A dimension is what you group by: text, dates, IDs. A metric is a number that gets aggregated.

The choice changes the numbers. Group by product name instead of SKU, and a product with three sizes becomes one row with its stock summed.

Formatting a field as Percent changes the report only. The BigQuery column does not change.

## Live charts

A chart can read from a saved query, so it updates as rows arrive. One practice dashboard read from a table being written by a streaming job. The streaming job is in `gcp-pipelines-and-orchestration`.

For anything many people open, point the report at a small summary table instead of the raw one. See `cost-and-table-design.md`.

## Console notes

- Chart types sit in their own row of the properties panel. The search box there searches settings, not charts.
- Written steps can claim to build an interactive filter while only adding a field. Check that a filter control actually exists.
