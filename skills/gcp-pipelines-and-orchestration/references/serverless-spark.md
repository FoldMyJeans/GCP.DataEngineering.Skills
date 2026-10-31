# Serverless Spark

Managed Service for Apache Spark (used to be Dataproc, and Serverless for Apache Spark). Spark splits work across many machines and runs code rather than SQL. Reach for it when there is existing Spark or Hadoop code to bring, or logic SQL cannot express. A plain `bq load` does a simple file load in one line.

## Cluster or no cluster

With a cluster you create machines, wait, run the job and delete the cluster. Serverless means you submit the job and Google finds the machines, so there is nothing to size, patch or forget to delete. Start with serverless unless something requires otherwise.

| Mode | Started by | For |
| --- | --- | --- |
| Batch | A `gcloud` command | Scheduled jobs |
| Interactive session | A notebook | Exploration |

```bash
gcloud dataproc batches submit pyspark my_job.py \
  --region=REGION \
  --deps-bucket=gs://my-bucket
```

`--deps-bucket` is where local files get uploaded, and is required when the job references files on your machine. Without `--subnet`, the job uses the `default` subnet in that region. The command still says `dataproc` under the new product name.

Not run here: that command as written. What was run is the template below.

## Running from a template

Google publishes Spark templates, so you pass settings and write no Spark. This one loaded an Avro file from a bucket into BigQuery:

```bash
export GCP_PROJECT=PROJECT_ID
export REGION=REGION
export GCS_STAGING_LOCATION=gs://PROJECT_ID
export JARS=gs://BUCKET/spark-bigquery-connector.jar

./bin/start.sh \
-- --template=GCSTOBIGQUERY \
    --gcs.bigquery.input.format="avro" \
    --gcs.bigquery.input.location="gs://PROJECT_ID" \
    --gcs.bigquery.input.inferschema="true" \
    --gcs.bigquery.output.dataset="my_dataset" \
    --gcs.bigquery.output.table="campaigns" \
    --gcs.bigquery.output.mode=overwrite \
    --gcs.bigquery.temp.bucket.name="PROJECT_ID-bqtemp"
```

Read it as a sentence: take Avro files from this bucket, work out the schema, write to this table, overwrite, and use that bucket as scratch.

`start.sh` reads the environment variables, not arguments. They live only in the terminal session where you set them, so a new window loses them.

## The setup it needs

```bash
gcloud compute networks subnets update default \
  --region=REGION --enable-private-ip-google-access
```

Private Google Access lets workers with no public IP reach Google services. The docs now say the service enables it on the subnet, but the practice run set it by hand, so check it if workers cannot reach a bucket.

Two buckets with different jobs:

| Bucket | Holds |
| --- | --- |
| Staging | The data and Spark's own files |
| Temp | Scratch space BigQuery needs while building the table |

Spark cannot write to BigQuery by itself. A connector library (a JAR file) teaches it how. Check the runtime docs for whether your version already includes it.

## Why Avro's schema mattered

Avro stores its schema in the file header, so the new table's columns came from the file and nobody typed them.

| Format | Layout | Best at |
| --- | --- | --- |
| CSV | Text rows, no types | Handing a file to a person |
| Avro | Row by row, binary, schema in the file | Data in motion: feeds, streams, messages |
| Parquet | Column by column, binary | Data at rest that will be queried |

## Which terminal

There are two, and they are easy to mix up. Cloud Shell is the small free machine every account gets. An SSH window belongs to a VM. The VM is only a workbench for files and commands, and it does no processing. Spark reads from the bucket, not from the VM, so copy the file into the bucket first.

## What breaks

**A warning about a missing metadata directory.** Harmless.

**"Batch job has failed" on the first try.** Waiting and running it again worked in the practice run. In a real project, read the batch logs before retrying.

**Where did the logs go?** The Spark history server and a metastore hold job history and table metadata after the short-lived machines are gone. Not run here: setting them up.

## Do not confuse

A Spark workflow template is a saved set of Spark jobs in order. A Dataflow template is a parameterized Beam pipeline. Workflows is a separate serverless orchestration product. Similar names, different products.
