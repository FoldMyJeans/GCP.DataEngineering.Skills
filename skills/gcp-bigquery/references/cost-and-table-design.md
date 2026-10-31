# Cost and table design

## What you pay for

On-demand BigQuery bills for the bytes a query reads, in the columns it touches. It does not bill for rows returned. Check the pricing page for the current rate. It was about 6.25 US dollars per TiB, with the first TiB each month free, when this was written.

Storage is separate. Clustering and materialized views lower query cost, not the storage bill. Storage cost drops with expiration, and with long-term pricing, which applies by itself to a table or partition left unmodified for 90 days. Reading does not reset that clock. Writing does.

| Lowers query cost | Does not |
| --- | --- |
| Selecting fewer columns | `LIMIT` on a non-clustered table |
| A filter on the partition column | `ORDER BY ... LIMIT`, which still reads the whole column |
| A filter on clustered columns | A filter on an unrelated column, for bytes read |
| Reading a small summary table | |

`SELECT *` is the biggest lever, because storage is by column and an unselected column is free. The Preview tab and `bq head` show rows at no charge.

The docs add one exception on `LIMIT`: on a clustered table it can reduce bytes read, because the scan stops once enough blocks are read. Do not count on it, and never for a non-clustered table.

## See the cost first

The console shows an estimate before you run. In one practice run it read 3.49 GB for a public table and 622 KB for a small one, so the gap is worth a glance.

```bash
# Estimate without running. Billed nothing
bq query --use_legacy_sql=false --dry_run 'SELECT name FROM my_dataset.names'

# Fail instead of spending more than this
bq query --use_legacy_sql=false --maximum_bytes_billed=1000000000 'SELECT ...'
```

The maximum works on the estimate before the query runs, so a query over the limit fails with no charge. On a clustered table, the docs warn that bytes cannot always be estimated accurately before running. A dry run against a table with row-level security always returns 0 bytes.

## Partitioning

Splits a table into slices. A query that filters on the partition column reads only the matching slices.

| Type | On |
| --- | --- |
| Time-unit column | A `DATE`, `TIMESTAMP` or `DATETIME` column, by hour, day, month or year. Hourly is not available on `DATE` |
| Ingestion time | When BigQuery received the row, through `_PARTITIONTIME` |
| Integer range | An integer column, with a start, end and interval |

```sql
CREATE TABLE sales.orders_by_day
PARTITION BY order_date
CLUSTER BY customer_id, country
AS SELECT * FROM sales.orders;
```

You cannot convert an existing non-partitioned table. Create a new one from it, as above, so decide this early.

Filter with a plain range. `EXTRACT(YEAR FROM ts) = 2024` can stop BigQuery skipping partitions:

```sql
WHERE ts >= '2024-01-01' AND ts < '2025-01-01'
```

## Clustering

Sorts rows inside the table by up to four columns, so a filter can skip blocks. Order matters: queries benefit most when they filter starting from the first clustered column.

You can add clustering to an existing table, but that does not re-sort the data already there. Only new data gets clustered automatically. The docs point to an `UPDATE` statement to recluster existing rows.

## Choosing between them

| Prefer | When |
| --- | --- |
| Partitioning | Queries filter on a date. You need a firm cost estimate before running. Partitions would average at least 10 GB. You want long-term pricing per partition |
| Clustering | The column has many distinct values. Queries filter on several columns. Each partition would hold about 10 GB or less |
| Both | Partition by date and cluster by what people filter on next. This is the common shape for a large table queried repeatedly |

Neither helps unless the query filters on that column.

Not run here: measuring the saving from partitioning and clustering on a large table. The practice runs created a partitioned table but did not compare bytes with and without the filter. Dry-run both versions on your own data.

## Views

A standard view stores only SQL text. It is always current and recomputes on every query, so it costs what its query costs each time.

A materialized view stores the result, and BigQuery refreshes it in the background and reads recent changes from the base table when needed. Queries against the base table can be rerouted to it automatically when it can answer them.

Not run here: materialized views. From the docs, the limits to check first:

- A restricted SQL syntax and a limited set of aggregation functions.
- You cannot change the query after creation, and you cannot nest one on another.
- Deleting a base table without deleting the view first makes queries and refreshes fail.
- Refresh cost is billed to the project where the view lives.

When the query does not fit those limits, a scheduled query into a table is the fallback. It is only as new as its last run.

## Process incrementally

Rebuilding all history every night is the expensive default. Append or merge only the new partition. Dataform's incremental tables do this as part of a pipeline. See `gcp-pipelines-and-orchestration`.

## Expiration

| Set on | Does |
| --- | --- |
| A partition | Drops old partitions and keeps the table |
| A table | Removes that table at a set time |
| A dataset default | Applies to tables created after it is set, not existing ones |

Sandbox tables expire after 60 days, which surprises people moving from practice to real work.
