# Dataform

SQL pipelines inside BigQuery, with version control. You write what each table is. Dataform works out the run order and BigQuery does the work.

## Repository and workspace

The repository is the git repo that holds the finished code. A workspace is your personal sandbox: a branch plus your uncommitted edits plus the editor. Nothing reaches the repository until you commit. Initializing a workspace creates the layout:

| Path | Holds |
| --- | --- |
| `definitions/` | `.sqlx` files |
| `includes/` | Reusable JavaScript |
| `workflow_settings.yaml` | Compilation settings, including default project and dataset |

A repository can link to an external git host, so you can edit `.sqlx` locally and review through pull requests. Compiling and running stay in the service. Not run here: linking an external repo.

## A SQLX file

SQL with a `config` block on top. The block decides what gets created, so you never write `CREATE OR REPLACE`.

```sql
config { type: "table" }

SELECT product, SUM(quantity) AS total_quantity
FROM ${ref("orders_view")}
GROUP BY product
```

`${ref("name")}` is the idea the tool exists for. It is a reference, not a table name, and writing it tells Dataform that the other action runs first. With that file open, the metadata pane lists the dependency, though nobody typed it anywhere else. In one run the table started 3 seconds after the view it depended on, with no order specified.

| `type` | Does |
| --- | --- |
| `declaration` | Points at an existing table, for your sources |
| `view` | Creates or replaces a view |
| `table` | Creates or replaces a table from a `SELECT` |
| `incremental` | Creates a table, then adds to it on later runs |
| `assertion` | A data quality test. Any row returned is a failure |
| `operations` | Custom SQL run as its own step |

A file can also have `pre_operations` and `post_operations` blocks around the main SQL, and shared logic goes in `includes/`.

## The service account

Dataform's service agent is `service-PROJECT_NUMBER@gcp-sa-dataform.iam.gserviceaccount.com`. It needs roles before anything runs:

```bash
SA=service-PROJECT_NUMBER@gcp-sa-dataform.iam.gserviceaccount.com
for ROLE in bigquery.jobUser bigquery.dataEditor bigquery.dataViewer; do
  gcloud projects add-iam-policy-binding PROJECT_ID \
    --member=serviceAccount:$SA --role=roles/$ROLE
done
```

| Role | Why |
| --- | --- |
| BigQuery Job User | Run queries at all |
| BigQuery Data Editor | Create and write tables |
| BigQuery Data Viewer | Read source data |

Grant these on the narrowest scope that works. The project-wide loop above came from a practice project. The docs also list column-level access roles when tables use policy tags.

Scheduled runs are different. The docs say a workflow configuration cannot run with the default service agent. It needs a custom service account or user credentials, and strict act-as is enforced, so the service agent also needs Service Account User and Service Account Token Creator on the custom account. Not run here: scheduled runs with a custom account. Read the Dataform access control docs when you set this up.

## Incremental tables

Not run here: this section comes from the docs. Try it on a scratch dataset first.

Rebuilding years of history every night is the expensive default. An incremental table adds only what is new.

```sql
config {
  type: "incremental",
  uniqueKey: ["transaction_id"],
  bigquery: {
    partitionBy: "DATE(timestamp)",
    updatePartitionFilter: "timestamp >= timestamp_sub(current_timestamp(), interval 24 hour)"
  }
}

SELECT timestamp, transaction_id, action
FROM ${ref("user_actions")}
${when(incremental(), `WHERE timestamp > (SELECT MAX(timestamp) FROM ${self()})`)}
```

- `when(incremental(), ...)` adds the filter on later runs only. The first run loads everything.
- `self()` refers to this table.
- `uniqueKey` turns the insert into a merge, so a row with a matching key is updated instead of added. Without it, a rerun can add the same rows again.
- `updatePartitionFilter` limits the partitions the merge scans. Set without `uniqueKey`, Dataform inserts instead of merging.

## Assertions

Built in, on the table they check:

```sql
config {
  type: "table",
  assertions: {
    uniqueKey: ["order_id"],
    nonNull: ["order_id", "customer_id"],
    rowConditions: ['order_date >= "2019-01-01"']
  }
}
```

Or written by hand, where any row returned is a failure:

```sql
config { type: "assertion" }

SELECT * FROM ${ref("orders")}
WHERE total < 0
```

Not run here: assertions. The trap, from the docs: **a failed assertion on table A does not stop table B that depends on A**. If B should wait for clean data, say so:

```sql
config {
  type: "table",
  dependencies: [{ name: "orders", includeDependentAssertions: true }]
}
```

`dependOnDependencyAssertions: true` does the same for every dependency of the file. Both default to false.

## Running on a schedule

| Path | Use when |
| --- | --- |
| Workflow configuration | Time-based, Dataform only. Needs a release configuration, a service account, a cron schedule and a time zone |
| Workflows with Cloud Scheduler | Dataform is one step among other API calls. The sample is in `scheduling-and-events.md` |
| Managed Airflow | Dependencies outside BigQuery. Uses `DataformCreateCompilationResultOperator` and `DataformCreateWorkflowInvocationOperator`, and needs Airflow 2 or later |
| Cloud Build trigger | Run after a git push |

Use tags to run part of a project, for example only the tables tagged `daily`. Not run here: all of these. The practice run was started by hand.

## What breaks

- The service account name is shown on the repository-created screen, and you leave that screen before the step that needs it. Copy it then.
- The quick "Grant all" button added only one of the three roles. Grant all three yourself.
- The create file dialog already contains `definitions/`. Type only the file name, or you get a folder inside a folder. A path typed with a colon creates a stray folder too.
- An error in the compiled queries pane before the roles are granted is expected.
- A new workspace comes with two sample view files. They also show up in the execution log. Delete or ignore them.
- Output lands in a dataset named `dataform` unless you change the settings. Set this before anything real runs.
