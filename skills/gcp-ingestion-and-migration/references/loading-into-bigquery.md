# Loading files into BigQuery

## bq load

A `gs://` path means BigQuery reads the file straight from Cloud Storage. Nothing downloads.

```bash
bq load \
  --source_format=CSV \
  --skip_leading_rows=1 \
  --schema=./trips_schema.json \
  my_dataset.trips \
  'gs://my-bucket/trips/*.csv'
```

| Flag | What it does | Watch out for |
| --- | --- | --- |
| `--source_format` | The file format | Defaults to CSV |
| `--skip_leading_rows=1` | Skips the header | The default is 0. With a schema file and a header in the CSV, leaving it out loads the header as data |
| `--schema=file.json` | Your own column list | Can be version controlled and reviewed, and it fails loudly when the source changes shape |
| `--autodetect` | BigQuery guesses from the header and sample rows | Header detection fails when every column is a string, so the header loads as a row |
| `--replace` | Erases existing data and schema, then loads | The default is false, which appends. `--noreplace` says so explicitly |
| Wildcard in the URI | Loads many files in one command | This is the shape of a real daily load: files land, one command sweeps them |

Appending by default is the trap in a rerun. Running the same load twice doubles the table, and nothing errors.

## LOAD DATA

The same job as SQL, so it can be scheduled, version controlled and run by a pipeline.

```sql
LOAD DATA INTO my_dataset.trips
FROM FILES (format = 'PARQUET', uris = ['gs://my-bucket/trips/*.parquet']);
```

`LOAD DATA OVERWRITE` replaces the table instead of appending.

Choose `bq load` for a person or a shell script. Choose `LOAD DATA` when the load is a step in a SQL pipeline.

## File formats

| Direction | Formats |
| --- | --- |
| Load into BigQuery | Avro, Parquet, ORC, CSV, JSON, Firestore exports |
| Export from BigQuery | CSV, JSON, Avro, Parquet |

ORC and Firestore exports go in but do not come out.

A JSON file holding one big array fails to load. BigQuery needs newline-delimited JSON, one object per line.

## Errors and silent failures

**`Not found: Dataset ...:my_dataset was not found in location US`.** The query ran in `US` while the dataset lives elsewhere. Pass `--location` to `bq`. In the console, open the dataset from the Explorer and start the query from there, which picks the right location.

**The bucket and dataset locations differ.** The bucket must be in the same location as the dataset you load into. Decide the location first.

**The file picker says "No buckets found."** It lists only buckets in your own project. For a shared bucket, type the path into the field.

**Row count is one too high.** The header loaded as data. Check for it with a `SELECT` on the odd row, then fix the load. Reload rather than patch when you can. If you must patch with a `DELETE`, run the same `WHERE` as a `SELECT` first.

**A numeric code lost its leading zero.** Autodetect read `01` as a number. Declare the column as `STRING` in a schema file and reload.

## After the load

```sql
SELECT COUNT(*) FROM my_dataset.trips;
```

Compare it with the source. Then look at the schema, because the type a column landed as is where the quiet damage shows up.
