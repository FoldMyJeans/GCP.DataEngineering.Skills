# SQL that runs and is wrong

An error is the good outcome, because you know to fix it. These run cleanly and return a wrong answer.

## The silent ones

**A missing comma makes an alias.** `AS` is optional, so two names in a row mean "first column, renamed to the second". You get one column with the wrong label.

```sql
-- Returns ONE column, customer_id, labeled order_total
SELECT customer_id order_total
FROM shop.orders;
```

Count the columns in the result.

**A join fans out.** If the other table has several rows per key, each left row repeats once per match, and a `SUM` of the repeated column comes out too high. Compare the row count before and after the join. Fix it by reducing the many side to one row per key before joining.

**`LIMIT` with no `ORDER BY` is a sample.** Tables have no built-in order, so each read may return different rows. If something must be repeatable, such as training data or a reconciliation, order it deterministically or save it as a table.

**`COUNT` counts rows, not things.**

| Write | Counts |
| --- | --- |
| `COUNT(*)` | Every row |
| `COUNT(col)` | Rows where `col` is not NULL |
| `COUNT(DISTINCT col)` | Each value once |

In a web table where quantity is NULL on rows with no purchase, `COUNT(quantity)` is orders and `SUM(quantity)` is units. An average per order divides by orders, not by all rows. Dividing by all rows gives units per view, which is a different number.

**Group totals can exceed the overall total.** Unique visitors per channel added up to more than total unique visitors, because one person can arrive through two channels. This one is correct, but it looks like a bug in a meeting.

**`UNION` only removes a row when the whole row matches.** If two lists share a name with different numbers, both rows stay, and with no label column you cannot tell them apart. Add a label column, or use a `JOIN` to put the numbers side by side.

**An alias is only a label.** Naming a count `product_views` does not make it count views. The filter does that.

**`ORDER BY` a name sorts alphabetically.** For biggest first, sort by the number.

**String matching is case-sensitive.** The same word with and without a capital comes back as two rows. Wrap the column in `LOWER()` to merge them.

**A join on dates stored as text can match nothing.** `2026-03-01` and `03/01/2026` as `STRING` never match, and you get zero rows with no error. Check the type in the Schema tab, because the values will not tell you, and fix it once in staging with `PARSE_DATE`.

## Duplicates

Find them on the key that should be unique. Zero rows back means the check passed.

```sql
SELECT order_id, COUNT(*) AS row_count
FROM shop.orders
GROUP BY order_id
HAVING row_count > 1
ORDER BY row_count DESC;
```

Then pull a few offenders sorted by key and timestamp, so the copies sit together and you can see how they differ. The cause decides the fix:

| Cause | Fix |
| --- | --- |
| A load ran twice and appended | Remove the copies, then make the load replace that day's data |
| Each version of an updated record was kept | Keep the latest per key |
| A join fanned out | Fix the join. Do not delete rows |
| Two genuine identical events | Nothing. They are not duplicates |

Keep the latest row per key. Leaving out `DESC` keeps the oldest instead:

```sql
SELECT *
FROM crm.customer_updates
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_at DESC) = 1;
```

For exact copies, compare `COUNT(*)` with `COUNT(DISTINCT TO_JSON_STRING(t))`. In a pipeline, automate the check. See `gcp-pipelines-and-orchestration`.

Before running a `DELETE` to clean up, run the same `WHERE` as a `SELECT`.

## Run order explains most rules

SQL runs in a different order from the one you write: `FROM` and joins, `WHERE`, `GROUP BY`, `HAVING`, window functions, `QUALIFY`, `SELECT`, `ORDER BY`, `LIMIT`.

| Filter on | Write | Why |
| --- | --- | --- |
| A row's value | `WHERE` | Runs before any sum exists |
| `COUNT` or `SUM` | `HAVING` | Runs after grouping |
| `ROW_NUMBER` or another window result | `QUALIFY` | Runs after window functions |

BigQuery lets `HAVING` and `ORDER BY` use an alias from the `SELECT`. `WHERE` never can.

A plain column next to an aggregate must be in the `GROUP BY` or inside an aggregate, because after grouping there is one row per group and the plain column would have several values for one cell.

## Window functions keep every row

`GROUP BY` collapses rows. A window function calculates across related rows and keeps them all. `PARTITION BY` inside `OVER()` only splits rows for that calculation, and it has nothing to do with table partitioning.

```sql
SELECT sale_date, daily_sales,
  SUM(daily_sales) OVER (ORDER BY sale_date) AS running_total,
  LAG(daily_sales) OVER (ORDER BY sale_date) AS prev_day
FROM sales.daily_totals;
```

`LAG` looks back and `LEAD` looks ahead. A scalar subquery that recomputes an aggregate scans the table a second time, and `QUALIFY` with a window function does it in one pass.

## Nested and repeated data

`{...}` in the JSON is a RECORD (also called STRUCT), one object with sub-fields. `[...]` is REPEATED (also called ARRAY), a list in one cell. Newline-delimited JSON, Avro, Parquet and ORC load these with an ordinary load job.

```sql
SELECT o.order_id, o.customer.city, item.sku, item.qty
FROM shop.orders AS o, UNNEST(o.items) AS item;
```

- `SELECT o.items.sku` fails, because a list has no single `sku`.
- The comma before `UNNEST` drops orders with an empty list. `LEFT JOIN UNNEST(o.items) AS item` keeps them.
- After `UNNEST`, `COUNT(*)` counts items, not orders.

A single JSON-type column also works, but values stay untyped and every query must extract and cast.

## Reading errors

| Error or symptom | Cause |
| --- | --- |
| Table name in square brackets with a colon fails | That is legacy SQL. GoogleSQL uses backticks and dots |
| `bq query` behaves oddly with GoogleSQL syntax | `bq` can default to legacy. Pass `--use_legacy_sql=false` |
| `bq` wants `project:dataset.table` | Colon in `bq` commands, all dots inside SQL |
| Column must appear in `GROUP BY` | A plain column next to an aggregate |
| Aggregate not allowed in `WHERE` | Move it to `HAVING` |
| Dataset not found in location | The query ran in the wrong location |
| Access Denied when running a query | Check both roles. See `access-and-recovery.md` |

When a project will not show in the Explorer, type the full table name in backticks in a query tab and Ctrl+click it. The table opens with its tabs, and the project then appears.

Fix one error at a time and read the message each time. Most broken queries have several problems stacked.

## Picking the right tool for logic

Use a SQL UDF for anything SQL can express. Use a JavaScript UDF when you need a library. Use a remote function, which calls Cloud Run over HTTP on every batch, only when you need real programming or an outside API.

Not run here: UDFs, stored procedures and remote functions. Check the BigQuery docs for current limits.
