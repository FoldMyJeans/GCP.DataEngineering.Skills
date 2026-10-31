# Querying files without loading them

Lakehouse tables used to be called BigLake tables. Newer docs and older docs both turn up, and both names appear in the console. It is the same product.

## Choose

| | Load into BigQuery | External table | Lakehouse table |
| --- | --- | --- | --- |
| Copies the data | Yes | No | No |
| Performance | High | Slower | High, with metadata caching |
| Column and row rules | Yes | No | Yes |
| Permissions users need | On the table | On the table and the data source | On the table only |
| Suited for | Everyday analytics | Occasional access | A governed data lake |

Neither kind of table over files gives you a cost estimate or a table preview, because BigQuery does not know what is in the bucket until it looks.

External tables can also read Google Sheets and Bigtable. A mapping sheet someone edits by hand, joined to a large fact table, is a good use: they keep editing, and the query picks up the changes with no reload.

## Why the connection is the whole point

Both tables point at files in Cloud Storage. The only structural difference is that a Lakehouse table names a connection. BigQuery reads the file through the connection's service account, so the person querying needs no access to the bucket.

That makes BigQuery the only door to the data, which is what lets it decide who sees which column. So one step is easy to miss:

**After moving people to Lakehouse tables, remove their direct Cloud Storage permissions.** If they can still open the file in the bucket, the column rules protect nothing, and nothing warns you.

## Setting it up

1. Create a connection resource. Its type is searched as Agent Platform (used to be Vertex AI). Search inside BigQuery's Add data panel, or use the Explorer tree: Connections, the three-dot menu, Create connection. Searching in the console's main search bar opens the Agent Platform product page instead.
2. Copy the connection's service account and grant it a Cloud Storage read role on the bucket.
3. Create the table. The create table form defaults to a native table, so switch it to External table and name the connection.
4. Add policy tags if you need column rules. The Add policy tag button appears only in Edit schema mode. The read-only Schema tab lets you tick fields but shows no button.

## Policy tags

A policy tag denies by default, even to a project owner. Access comes from a separate role granted on the tag: Fine-Grained Reader (`roles/datacatalog.categoryFineGrainedReader`) for real values, or Masked Reader for masked values.

Once columns are tagged, `SELECT *` fails for anyone without the role. Leaving the tagged columns out works:

```sql
SELECT * EXCEPT(address, phone, postal_code)
FROM my_dataset.customers;
```

Not run here: whether tags survive a schema change. Check the BigQuery docs before changing the schema of a tagged table, and check the tags again afterward. The upgrade below replaces the table's definition, so check them after that too.

## Upgrading an external table in place

No rebuild and no reload. Three commands:

```bash
# A definition that says: these files, read through this connection
bq mkdef --source_format=CSV --connection_id=PROJECT.LOCATION.CONNECTION \
  'gs://my-bucket/customers.csv' > table_def.json

# Save the current column list
bq show --schema --format=prettyjson my_dataset.customers > schema.json

# Apply both
bq update --external_table_definition=table_def.json --schema=schema.json \
  my_dataset.customers
```

The saved schema matters because `bq update` replaces the whole definition. Without it, auto detect would guess the types again.

Prove it worked: the table details show a connection ID under the external data configuration, and the table is marked as a Lakehouse table.

## Metadata cache

It stores file sizes, row counts and column statistics, so BigQuery can skip listing the bucket and prune files without opening them. A query for one day skips any file whose latest date is earlier.

Staleness is configurable from 30 minutes to 7 days, with automatic or manual refresh. Caching is off by default on external tables, and BigQuery shows a banner about it after a query.
