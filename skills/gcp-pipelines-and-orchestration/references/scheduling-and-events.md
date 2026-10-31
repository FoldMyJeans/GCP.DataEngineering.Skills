# Scheduling and events

Not run here: everything in this file. The practice projects started their jobs by hand. The commands below come from the Google Cloud docs, so check them before relying on them, and test each on a scratch resource.

## Scheduled queries

One SQL statement on a timer, with a destination table. `WRITE_TRUNCATE` overwrites and `WRITE_APPEND` adds. `@run_time` (a UTC timestamp) and `@run_date` (a date) are available as parameters, and using them is how a rerun stays pinned to its own day.

They run with user credentials by default. Select a service account instead, so the schedule does not depend on one person's account.

A scheduled query runs one statement and stops. It cannot trigger the next step or test its output. Once you need that, move to Dataform.

## Cloud Scheduler and Workflows

Scheduler is a clock. It fires an HTTP call, a Pub/Sub message or a Workflows execution, and does no work itself. Workflows is the list of steps: serverless, no charge while idle, with HTTP retries and error handling, and able to wait for up to a year. Together they are the cheap answer for a few steps in order.

Starting a Dataform run takes two calls: compile the code, then invoke the result. From the Dataform docs:

```yaml
main:
  steps:
  - init:
      assign:
      - repository: projects/PROJECT_ID/locations/LOCATION/repositories/REPOSITORY_ID
  - createCompilationResult:
      call: http.post
      args:
        url: ${"https://dataform.googleapis.com/v1/" + repository + "/compilationResults"}
        auth:
          type: OAuth2
        body:
          gitCommitish: main
      result: compilationResult
  - createWorkflowInvocation:
      call: http.post
      args:
        url: ${"https://dataform.googleapis.com/v1/" + repository + "/workflowInvocations"}
        auth:
          type: OAuth2
        body:
          compilationResult: ${compilationResult.body.name}
          invocationConfig:
            includedTags:
            - daily
            transitiveDependenciesIncluded: true
      result: workflowInvocation
  - complete:
      return: ${workflowInvocation.body.name}
```

The workflow's service account needs Dataform Editor, plus Service Account User on the custom account Dataform runs as.

## Managed Airflow

Managed Service for Apache Airflow (used to be Cloud Composer) is hosted Airflow, and your DAG files are plain Airflow Python. A DAG (directed acyclic graph) is tasks plus the order they run in. Dataform infers its graph from SQL, and in Airflow you write the arrows yourself.

```python
load = GCSToBigQueryOperator(...)
join = BigQueryInsertJobOperator(...)
process = DataprocSubmitJobOperator(...)

load >> join >> process
```

| Word | Means |
| --- | --- |
| Operator | A template for one kind of action |
| Task | One use of an operator, with its parameters filled in |
| Dependency | Which task must finish first |

Deploy by putting the file in the environment's DAGs folder in its Cloud Storage bucket.

Check the pricing page before creating an environment. It runs as a standing set of components, so a schedule of two steps does not justify it. Use it for many tasks, dependencies across products, systems outside Google Cloud, backfills (re-running past dates that were missed), or a team that already writes Airflow.

Generations differ: only Gen 3 supports Airflow 3, and Dataform's operators need Airflow 2 or later.

## Eventarc and Cloud Run functions

An event-driven pipeline starts when something happens. Choose it when files arrive at unpredictable times, and choose a schedule when they arrive at known times.

Eventarc is the router and does not run code. A Cloud Run function (used to be Cloud Functions) is the small piece of code that does. It should only start the real work: functions are for short glue code, so hand the heavy part to BigQuery, Dataflow or Spark.

```bash
gcloud eventarc triggers create file-landed \
  --location=REGION \
  --destination-run-service=my-service \
  --destination-run-region=REGION \
  --event-filters="type=google.cloud.storage.object.v1.finalized" \
  --event-filters="bucket=my-bucket" \
  --service-account=SA_NAME@PROJECT_ID.iam.gserviceaccount.com
```

Three requirements that are easy to miss:

- The bucket must be in the same project and the same region or multi-region as the trigger.
- The Cloud Storage service agent needs Pub/Sub Publisher (`roles/pubsub.publisher`).
- The trigger's service account needs Cloud Run Invoker (`roles/run.invoker`) for an authenticated service.

BigQuery has no "when a row is inserted" trigger of its own. The audit log is the side door, and Eventarc can watch it, so almost anything that happens in Google Cloud can be a trigger.

A trigger can deliver an event more than once, so a job it starts should be safe to run twice.

## Watching pipelines

| Question | Tool |
| --- | --- |
| Tell me when it fails or falls behind | Cloud Monitoring alerting policy |
| Show the trend | Cloud Monitoring dashboard |
| What was the exact error? | Cloud Logging, Logs Explorer |

A pipeline with no alert fails silently until someone notices the dashboard is stale. Set the alert when you build the pipeline.
