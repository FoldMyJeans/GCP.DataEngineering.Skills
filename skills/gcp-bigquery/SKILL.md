---
name: gcp-bigquery
description: >-
  Write BigQuery SQL that is actually right, design tables so queries stay cheap, and control and
  recover the data. Covers silent SQL failures (missing comma, join fan-out, LIMIT with no ORDER
  BY, COUNT vs COUNT DISTINCT), duplicates and QUALIFY, nested data and UNNEST, partitioning and
  clustering, views vs materialized views vs scheduled tables, dry runs and cost, IAM roles,
  policy tags, row-level security, authorized views, time travel, snapshots, clones, and reports
  in Data Studio (used to be Looker Studio). Use whenever the user writes or fixes a BigQuery
  query, has a wrong total or a surprising bill, designs a table, grants access, needs to undo a
  change, or connects a dashboard. Trigger on "my query is wrong", "numbers don't match",
  "duplicates", "partition this table", "cluster", "materialized view", "why did this query cost
  so much", "who can see this table", "hide this column", "restore a deleted table", "UNNEST",
  "legacy SQL", "Access Denied" on a query.
---

# GCP BigQuery

This skill covers working with data once it is in BigQuery: querying it, shaping tables, controlling access, recovering from mistakes, and reporting on it. Getting data in is `gcp-ingestion-and-migration`. Chaining SQL into a pipeline is `gcp-pipelines-and-orchestration`. Training models in SQL is `gcp-ml-and-agents`.

It starts from hands-on practice projects. Advice that was not run hands-on is labelled "Not run here:", so check it against the current docs before relying on it.

## Before writing a query

1. **Say what one row of the table is.** One hit, one order, one order line. In a web analytics table one row was roughly one page hit, so 21 million rows was nowhere near 21 million visits.
2. **Finish the sentence "one row in my result is one ...".** The words after "per" or "each" in the question are the `GROUP BY` columns.
3. **Look at the table first.** The Schema, Details and Preview tabs are free, and most wrong answers start with a wrong guess about a column. `bq show` and `bq head` do the same from a terminal.

## Choose how to serve a result

| The need | Use | Not |
| --- | --- | --- |
| Always current, read rarely, or hiding columns | Standard view | A table that goes stale |
| Current and fast, same aggregation read all day | Materialized view | A standard view, which reruns every time |
| Heavy report that can be a little old, or SQL a materialized view cannot express | Scheduled query into a table | Letting every dashboard scan the raw table |
| A frozen copy of a result | `CREATE TABLE ... AS SELECT` | Expecting it to stay in sync. It quietly goes stale |

Aggregate once, read many times. One job writes a small summary table and the dashboards read that, instead of each viewer scanning the raw table.

## Choose how to protect data

| Hide what | Use |
| --- | --- |
| A whole dataset from a person | Grant nothing on it. Grant readers on the dataset, not the project |
| One column, keeping the table usable | Policy tag |
| Part of a value, such as the last four digits | Dynamic data masking on a policy tag |
| Some rows, by who is asking | Row-level access policy |
| Everything except a query result | Authorized view |
| Files in a bucket, with the same column rules | Lakehouse table. See `gcp-ingestion-and-migration` |

## Choose how to undo

| What happened | Fix |
| --- | --- |
| Bad change or deleted table within the time travel window | Time travel |
| Need a known good copy kept longer | Table snapshot, taken before the change |
| Need a writable copy to test on | Table clone |
| Past the window | Fail-safe, through Cloud Customer Care. You cannot read it yourself |

## Rules of thumb

- **A query that runs is not a query that is right.** The worst failures here raise no error: a missing comma, a fan-out join, a `COUNT` that should be `COUNT(DISTINCT ...)`. See `references/sql-silent-failures.md`.
- **`LIMIT` is not a cost control, and without `ORDER BY` it is not a fixed subset.** A view defined with `LIMIT` and no `ORDER BY` returned different rows each time it was read, and two evaluations of "the same" view counted 85 and 146 buyers.
- **Use `SAFE_CAST` for dirty values.** It returns NULL on a bad value. A made-up 0 gets counted in averages and skews them, and NULL is skipped.
- **Check on the business key, not on all columns.** If the key has no duplicates, the full rows cannot have any. The reverse is false: two rows with the same key and one different value pass an all-columns check.
- **Ask why duplicates exist before deleting them.** A retried load, a kept update history and a bad join each need a different fix.
- **Question extreme values.** The top rows of one "heaviest" result all had the identical value, which is a cap or a placeholder. Check units too: one source stored money multiplied by a million.
- **Filter on the partition column with a plain range.** Wrapping it in a function can stop BigQuery skipping partitions.
- **Running a query takes two roles in two places.** Job User on the project that runs it, Data Viewer on the dataset that holds the data.
- **Grant the smallest role on the smallest resource.** Data Viewer on the project reads every dataset in it.
- **Set the location before you start.** A "dataset was not found in location US" error means the query ran in the wrong location. Details are in `gcp-ingestion-and-migration`.

## Before you hand this back

- The result was checked against something outside the query: a source total, a known number, a reconciliation.
- Row count before and after each join compared, so fan-out would show.
- Tables queried repeatedly are partitioned or clustered on the columns people filter by.
- A dry run was done before any large or repeated query.
- No project-level grant where a dataset-level grant would do, and no Basic role on production data.

## References in this skill

- `references/sql-silent-failures.md`: SQL that runs and is wrong, duplicates, run order, nested data, and how to read an error.
- `references/cost-and-table-design.md`: what you pay for, partitioning, clustering, materialized views, and dry runs.
- `references/access-and-recovery.md`: IAM roles, policy tags, row-level security, authorized views, time travel, snapshots and clones.
- `references/reporting-on-bigquery.md`: Data Studio (used to be Looker Studio), credentials, dimensions and metrics, and when to use Looker.
