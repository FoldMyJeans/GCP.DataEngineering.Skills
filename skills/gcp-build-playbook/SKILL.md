---
name: gcp-build-playbook
description: >-
  Plan and run a whole data project on Google Cloud end to end, and route to the right skill at
  each phase. Covers the phase order (questions, sources, project and location, landing data,
  transforming, quality checks, serving, scheduling, hardening), the decisions that are expensive
  to reverse, the service account pattern, choosing storage (BigQuery, Cloud SQL, Spanner, Bigtable,
  Firestore, Cloud Storage), and working with a console that no longer matches written steps. Use
  whenever the user is starting a data platform, pipeline or warehouse on GCP, asks what to build
  first or in what order, asks which product or storage to use, hits a permission error from a
  service, or asks which of these skills applies. Trigger on "build a data pipeline on GCP", "new
  data project", "where do I start", "what order", "which database should I use", "plan this
  build", "which skill", "design the architecture", "the button is not there", "the steps don't
  match the console", "permission denied on a service account".
---

# GCP build playbook

The order to build a Google Cloud data project in, and which skill owns each phase. Use this when starting something new, or when you are not sure which skill answers a question.

It starts from hands-on practice projects, not production systems. Advice that was not run hands-on is labelled "Not run here:", and that includes the overall phase order here. Check specifics against the current docs before relying on them.

## Which skill answers this

| The question | Skill |
| --- | --- |
| How does the data get in, how do I load or replicate it | `gcp-ingestion-and-migration` |
| Why is this query wrong or expensive, how should the table look, who can see it | `gcp-bigquery` |
| How do I transform it, stream it, schedule it or chain the steps | `gcp-pipelines-and-orchestration` |
| How do I predict, classify, call Gemini or build an agent | `gcp-ml-and-agents` |

## The phase order

### 1. Write down the questions

List what the data must answer, in the business's words. That decides the tables. For each output, say what one row is.

### 2. Inventory the sources

For each source: files or a live database, once or continuous, how much, who owns it, and what you will need to ask them for. Getting a source ready is usually a request to someone else, and it often has the longest lead time in the project.

### 3. Set up the project and decide the location

Separate projects for dev and prod. A dedicated service account per job. Decide the location now, before creating any bucket or dataset, because a load needs the bucket and the dataset in the same location, and moving later means copying everything.

### 4. Land the raw data

Files land in a bucket, then load to a raw dataset. Keep raw data untouched. With ELT, a bug in your logic means re-running SQL on data you already have, and with ETL it means extracting two years of history again, if it still exists. See `gcp-ingestion-and-migration`.

Mask sensitive fields before landing only when compliance requires it, or leave them out of the replication altogether.

### 5. Transform

Raw to staging to curated, in SQL, in version control. Per-source code only conforms to a common shape. Business logic is written once, after the union. See `gcp-pipelines-and-orchestration`.

### 6. Check quality in the pipeline

Row counts, a key check for duplicates, ranges, and a reconciliation against the system of record. A total that matches finance convinces people more than any SQL. Make downstream steps wait for the checks. See `gcp-bigquery` and `gcp-pipelines-and-orchestration`.

### 7. Serve it

Small summary tables for dashboards, access granted at the dataset level, and column rules where needed. Get one report working end to end before building ten. See `gcp-bigquery`. If a model is wanted, see `gcp-ml-and-agents`.

### 8. Schedule it

Only now, and with the smallest tool that covers the job. Run it by hand until a rerun is known to be safe. See `gcp-pipelines-and-orchestration`.

### 9. Harden it

Alerts on failure, least privilege, no person's login in any job, and an owner for everything that bills while idle.

## Expensive to reverse

Make these on purpose, early.

| Decision | Why it hurts later |
| --- | --- |
| Location of buckets and datasets | They must match for a load. Moving means copying |
| Partitioning of a large table | An existing table cannot be converted, so you rebuild it |
| What one row is | Every query and report depends on it |
| Keeping raw data | Thrown away, it may be gone for good |
| A person's login running a job | The job breaks when they leave, and it is hard to find |

## Choose storage

Ask two things: what shape is the data, and is the work many small writes or few huge reads?

| The need | Use |
| --- | --- |
| Files of any kind, a data lake | Cloud Storage |
| Questions across many rows | BigQuery |
| An app saving one record at a time | Cloud SQL |
| Relational, global scale, five nines | Spanner |
| PostgreSQL with heavy mixed work | AlloyDB |
| A flood of timestamped writes, lookup by key in milliseconds | Bigtable |
| App documents with live sync to devices | Firestore |

Cloud SQL is the cash register and BigQuery is the accountant. Real systems have both, with data copied from one to the other. Never make BigQuery the backend of a website.

## The pattern to recognize

**Create a service account, give it roles, and it does the work.** Datastream, Lakehouse connections, Dataform, remote models and scheduled jobs all follow this.

When a service fails with a permission error, first ask which service account is acting, then what roles it has and where they were granted. Do not start with your own roles. Google Cloud also has service agents, which Google creates and you grant roles to. Their names look like `service-PROJECT_NUMBER@gcp-sa-SERVICE.iam.gserviceaccount.com`.

## Rules of thumb

- **"Created" does not mean "working".** Look for the proof: rows in the destination, a running worker, a connection ID on the table.
- **No error does not mean correct.** Check row counts after every load.
- **Stop what bills.** Streaming jobs, deployed models, database instances and Airflow environments keep charging while idle.
- **Pick the simplest thing that meets every requirement.** A product's own scheduler before an orchestrator, `bq load` before Spark, SQL before Dataflow.
- **Ask for everything from a source owner at once.** Each round trip can cost days.
- **Trust the output over the written steps.** See `references/working-in-the-console.md`.

## References in this skill

- `references/working-in-the-console.md`: when steps do not match the screen, sign-in traps, slow resources, and practice project limits.
