---
name: gcp-pipelines-and-orchestration
description: >-
  Choose and build data pipelines on Google Cloud: Dataform SQL pipelines and assertions, Dataflow
  batch and streaming jobs and templates, Pub/Sub, BigQuery subscriptions, serverless Spark on
  Managed Service for Apache Spark (used to be Dataproc), Dataprep, Data Fusion, and what runs them:
  scheduled queries, Cloud Scheduler, Workflows, Managed Service for Apache Airflow (used to be
  Cloud Composer), Eventarc and Cloud Run functions. Use whenever the user builds a transform,
  chains SQL steps, streams events into BigQuery, runs a Spark job, schedules or orchestrates
  work, or reacts to a file landing. Also use for a Dataflow job stuck at zero workers, a pipeline
  that keeps billing, or a failed assertion. Trigger on "ETL", "ELT", "build a pipeline",
  "Dataform", "SQLX", "Dataflow", "streaming into BigQuery", "Pub/Sub", "Spark job", "Airflow",
  "Composer", "DAG", "schedule this", "run when a file lands", "dead letter".
---

# GCP pipelines and orchestration

This skill covers transforming data and deciding what runs the work. Getting data into Google Cloud in the first place is `gcp-ingestion-and-migration`. Whether the SQL inside a step is right is `gcp-bigquery`. ML pipelines are `gcp-ml-and-agents`.

It starts from hands-on practice projects. Advice that was not run hands-on is labelled "Not run here:", so check it against the current docs before relying on it.

## Two separate choices

Pick the engine that does the work, then pick what starts it. Airflow is not an alternative to Dataflow: Airflow decides when and in what order, and Dataflow is one engine it can call.

## Choose the engine

Ask where the compute should happen. Inside the warehouse is ELT, and that is the default. Elsewhere first is ETL, which earns its place when data must change before it lands, when the logic is not SQL, or when it is streaming.

| The job | Use | Not |
| --- | --- | --- |
| SQL transforms inside BigQuery, one statement | Scheduled query | A pipeline product |
| SQL transforms inside BigQuery, chained, with tests and git | Dataform | A pile of scheduled queries |
| Events or a stream, or logic SQL cannot express, or a new pipeline in code | Dataflow | Two separate tools for batch and streaming |
| Existing Spark or Hadoop code, or a team that writes Spark | Serverless Spark on Managed Service for Apache Spark (used to be Dataproc) | A rewrite in Beam |
| A person cleaning one messy file with no code | Dataprep (batch only) | Dataflow |
| Many systems wired visually, hybrid or multicloud | Cloud Data Fusion (not serverless, runs on Spark) | Dataflow, which is code or templates |
| Events into BigQuery, unchanged | Pub/Sub BigQuery subscription | A Dataflow job with nothing to do |

Dataform, Dataflow and Datastream share letters and nothing else. Dataform is SQL files in git. Dataflow is the Beam engine. Datastream is a database change log pipe. Google's Dataflow is also unrelated to Power BI's dataflows.

**A row changed in a database is Datastream. An app emitted an event is Pub/Sub.** Clicks and logs never touch a database, so `site -> Pub/Sub -> Dataflow -> BigQuery`. See `gcp-ingestion-and-migration` for Datastream.

## Choose what starts it

Pick the smallest thing that covers the whole job. Managed Airflow can do all of these, which is why it looks like the safe pick, and why it is usually overkill.

| The need | Use |
| --- | --- |
| One SQL statement on a timer | Scheduled query |
| Chained SQL tables on a timer | Dataform workflow configuration |
| A clock for any HTTP target | Cloud Scheduler |
| A few service calls in order, serverless | Workflows, started by Cloud Scheduler |
| Many products, dependencies, backfills, or an existing Airflow team | Managed Service for Apache Airflow (used to be Cloud Composer) |
| Run as soon as a file lands | Eventarc to a Cloud Run function (used to be Cloud Functions) |
| Many steps after an event | Eventarc to Workflows |

A job that checks a bucket every minute is polling, not event-driven: it runs when nothing arrived and reacts late. Scheduler is only a clock. Eventarc only routes. **The trigger is not the worker**: a function catches the event and hands off the heavy work.

## Rules of thumb

- **"Created" does not mean "working".** A Dataflow job said Running while its worker count stayed at 0 and its graph turned to Failed. The proof is a current worker and no red log lines.
- **Stop what bills.** A streaming job never ends by itself and bills until you stop it. Stopping it is a step of its own, so write it down when you start one.
- **The service account does the work.** Dataform, Dataflow and Spark all act as a service account. When something fails on permissions, ask which account is acting.
- **Assertions do not stop anything by default.** A failed Dataform assertion on a table does not block the table downstream unless you make it a dependency. See `references/dataform.md`.
- **A dead-letter topic is not an alert.** It keeps messages Pub/Sub could not deliver. Telling a person is a Cloud Monitoring alert. A message Dataflow received and your checks rejected never fails in Pub/Sub, so write those to an error table.
- **Do not patch bad source data in the pipeline.** Silently defaulting or dropping hides the problem. Send those rows to a quarantine output and report them to whoever owns the source.
- **Make a rerun safe.** A load that fails halfway, gets re-run and appends leaves that day's rows doubled, with no error. A step should replace its own day's data, or merge on a key.
- **Per-source code stays thin.** It only conforms to a common shape: rename, cast, convert. Business logic is written once, after the union.

## Before you hand this back

- A second run of the pipeline produces the same result, with no doubled rows.
- Something tells a person when it fails: an alert, not just a log line.
- Quality checks exist on the outputs, and downstream steps actually wait for them.
- Every streaming job, cluster and Airflow environment has an owner and a plan to stop it.
- It runs as a dedicated service account with only the roles it needs, not a person's login.

## References in this skill

- `references/dataform.md`: SQLX, `ref()`, incremental tables, assertions, service account roles, and scheduling.
- `references/dataflow-and-pubsub.md`: templates, streaming shape, UDFs, dead letters, stopping a job, and the zero worker failure.
- `references/serverless-spark.md`: batches, templates, the BigQuery connector, the two buckets, and which terminal to use.
- `references/scheduling-and-events.md`: scheduled queries, Scheduler with Workflows, Airflow DAGs, Eventarc triggers, and monitoring.
