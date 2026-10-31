# Datastream: PostgreSQL to BigQuery

You only ever write to PostgreSQL. BigQuery fills itself in. That is the test of whether it works.

## The parts

Two connection profiles and one stream. A profile is only saved connection details and does nothing by itself. The stream is the job that uses both.

| | Source profile | BigQuery profile |
| --- | --- | --- |
| Needs host, port, user, password | Yes | No |
| Why | A separate database with its own login | Same project, so permission comes from IAM |

## Prepare the database

**1. Turn on logical decoding.** It is off by default, and there is no Datastream without it. On Cloud SQL, set the flag and restart the instance:

```bash
gcloud sql instances patch my-instance \
  --database-flags=cloudsql.logical_decoding=on
```

`patch` with `--database-flags` replaces the instance's whole flag list, so include any flags already set.

**2. Allow the connection.** A new Cloud SQL instance accepts connections from nobody. For IP allowlisting, add every published Datastream address for your region. Datastream runs on a fleet, so listing one address works until Google switches machines. The list differs by region, so look it up each time. Never copy it from old notes.

**3. Create the replication objects**, in the database being replicated:

```sql
-- What may be read. Name the tables, which puts less load on the database than FOR ALL TABLES.
CREATE PUBLICATION my_publication FOR TABLE sales.orders, sales.order_lines;

-- Who may read. Use a dedicated account, never a person's login.
CREATE USER datastream_user WITH REPLICATION LOGIN PASSWORD '<password>';
GRANT USAGE ON SCHEMA sales TO datastream_user;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO datastream_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA sales GRANT SELECT ON TABLES TO datastream_user;

-- How far the reader got. Each stream needs its own slot.
SELECT PG_CREATE_LOGICAL_REPLICATION_SLOT('my_slot', 'pgoutput');
```

Publication and slot have confusingly similar names, and both get typed into the stream form. The publication is the list of tables. The slot is a bookmark in the change log. If the stream is offline for an hour, the slot makes the database keep the unread changes, so nothing is lost.

**4. Check replica identity.** `REPLICA IDENTITY DEFAULT` identifies rows by primary key in the change log, and updates and deletes need that. A table with no primary key needs a decision before you start.

## Create the stream

Set the slot, publication and schemas on the source side, and the staleness limit on the BigQuery side. Staleness is the cost versus freshness setting. Zero applies changes right away instead of batching them. Run the connection test on the profile and the validation on the stream before starting.

Replicate selectively. You can pick schemas, tables and columns, and a sensitive column left out never enters Google Cloud.

## Prove it works

1. Existing rows appear in BigQuery with nobody asking. That is the backfill.
2. Insert, update and delete in PostgreSQL. BigQuery follows, within the staleness limit. That is change data capture.
3. The BigQuery table has the source columns plus a `datastream_metadata` record column.

The Preview tab can say "There is no data to display" while rows are arriving, because it cannot see rows still streaming in. Run a `SELECT` instead.

## What breaks

**The password prompt fails at once, before you can type.** Leftover line breaks from an earlier multi-line paste got read as an empty password. Press Enter a few times and retry.

**The connection test passes, then the stream fails with a permissions error.** Not run here: this is the failure to expect when the account was created without `REPLICATION`. Ask for it explicitly.

**The disk fills on the source.** A stream that was stopped and never deleted leaves its slot asking the database to keep changes forever. Delete the stream and the slot together. Long-running transactions and large batch operations grow the log as well. Cloud SQL and AlloyDB do not support `max_slot_wal_keep_size`, so the Google docs suggest `statement_timeout` as a precaution there.

**A `TRUNCATE` on the source does not reach BigQuery.** Datastream does not replicate it, so the destination keeps the rows and nothing errors. Watch for this with teams that clear tables as part of a batch job.

**More than 10 streams, or other tools using logical replication.** Raise `max_replication_slots` and `max_wal_senders`. `max_replication_slots` takes effect only at server start, so plan the restart.

## Asking for this in a real job

The form is the easy part. Getting the source ready is usually a request to someone else, so ask for all of it at once:

- A dedicated account with `REPLICATION` and read access.
- Logical decoding enabled, with a restart window.
- The publication and slot created, or permission to create them.
- The published Datastream addresses allowlisted, for the region in use.
- SSL. An unencrypted connection is a practice shortcut.

Not run here: private connectivity instead of IP allowlisting, which is what a production network will usually want. Read the Datastream connectivity docs.

Other engines use a different log mechanism: binary log on MySQL, LogMiner on Oracle, transaction logs on SQL Server. The setup above is PostgreSQL only.
