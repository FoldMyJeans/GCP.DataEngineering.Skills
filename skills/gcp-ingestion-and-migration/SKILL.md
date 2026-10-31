---
name: gcp-ingestion-and-migration
description: >-
  Choose how data gets into Google Cloud, then set it up: gcloud storage, Storage Transfer
  Service, Transfer Appliance, bq load, LOAD DATA, BigQuery Data Transfer Service, Database
  Migration Service, Datastream change data capture, external tables and Lakehouse (used to be
  BigLake) tables. Use whenever the user wants to load a file into BigQuery, move data from
  on-premises or Amazon S3 or Azure, replicate a PostgreSQL or MySQL or Oracle or SQL Server
  database, keep BigQuery in sync with a source, query files in a bucket without loading them,
  or plan a migration. Also use for a load with wrong types or wrong row counts, a "dataset was
  not found in location" error, or a Datastream stream that is failing or filling the source
  disk. Trigger on phrases like "load this CSV into BigQuery", "replicate Postgres to BigQuery",
  "move this data from S3", "CDC", "change data capture", "bq load", "Datastream", "external
  table", "how do I get this data into GCP", "transfer appliance".
---

# GCP ingestion and migration

This skill covers getting data into Google Cloud. It stops once the data has landed. Querying it is `gcp-bigquery`. Transforming, scheduling and orchestrating it are `gcp-pipelines-and-orchestration`.

It starts from hands-on practice projects. Advice that was not run hands-on is labelled "Not run here:", so check it against the current docs before relying on it.

## Choose the tool first

Ask two questions: is this files or a live database, and does it happen once, on a schedule, or continuously?

| The job | Use | Not |
| --- | --- | --- |
| Copy files to a bucket, by hand, once | `gcloud storage cp` | `gsutil`, the legacy tool for the same job |
| Copy files unattended or on a schedule, or from S3 or Azure | Storage Transfer Service | A cron job around `gcloud storage` |
| The network cannot finish in reasonable time | Transfer Appliance | Waiting. See the bandwidth check below |
| Bucket file into a BigQuery table, by hand or script | `bq load` | |
| Bucket file into a BigQuery table, from a SQL pipeline | `LOAD DATA` | |
| SaaS, object store or other warehouse into BigQuery, on a schedule, no code | BigQuery Data Transfer Service | Storage Transfer Service, which only delivers to Cloud Storage |
| Every insert, update and delete must reach BigQuery | Datastream | A nightly full extract |
| Move a database workload to Cloud SQL or AlloyDB | Database Migration Service | Datastream, which feeds analytics destinations |
| Query files where they sit, occasionally, no column rules | External table | Loading it |
| Query files where they sit, with column or row rules | Lakehouse (used to be BigLake) table | External table, because users can go around it through the bucket |

The two transfer services are the classic mix-up, and the names do not help. Storage Transfer Service moves files into Cloud Storage. BigQuery Data Transfer Service moves structured data into BigQuery.

**Bandwidth check.** 1 TB takes about a day over 100 Mbps and about 2 minutes over 100 Gbps. Work this out before choosing. Hundreds of terabytes on a slow link means months online, which is when the appliance earns its shipping time.

## Rules of thumb

- **A load with no error is not a correct load.** Check the row count after every load. A CSV imported into Cloud SQL once counted 955 rows instead of 954: the header loaded as a row, and the text `num` in an integer column silently became 0.
- **Autodetect is for exploring.** Use a schema file for anything that runs twice. A code like `01` read as a number loses its leading zero, and nothing errors.
- **Prefer Parquet or Avro over CSV when you control the export.** They carry their own types, so a date stays a date.
- **Keep the bucket and the dataset in the same location.** BigQuery requires it for a load. Decide the location before creating either one.
- **The service account does the work.** Datastream, Lakehouse connections and Dataform all follow it: create or find the service account, grant it roles, and let it act. When something in this area fails with a permission error, ask which service account is acting before you look at the person's own roles.
- **"Created" does not mean "working".** Look for the proof: rows in the destination, a running worker, a connection ID on the table.
- **A replication slot needs an owner.** A stream that is stopped and left behind keeps its PostgreSQL slot, and the database holds changes for it until the disk fills. Delete the stream and the slot together.
- **Written steps go stale fast, because the console moves.** When a button is missing, a `gcloud` or `bq` command usually does the same job.

## Before you hand this back

- Row counts checked at source and destination.
- Schema reviewed, with types and leading zeros looked at, not just "the load passed".
- No practice shortcuts left in: no person's login, no unencrypted database connection, no broad role granted to save time.
- Anything that bills while idle or fills a disk has an owner.

## References in this skill

- `references/loading-into-bigquery.md`: `bq load`, `LOAD DATA`, the flags that matter, file formats, and load errors.
- `references/datastream-postgres.md`: preparing PostgreSQL, the stream's parts, how to prove it works, and what breaks in real use.
- `references/query-without-loading.md`: external tables vs Lakehouse tables, the service account delegation, and upgrading a table in place.
