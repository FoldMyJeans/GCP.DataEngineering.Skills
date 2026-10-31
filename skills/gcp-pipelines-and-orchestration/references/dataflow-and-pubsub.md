# Dataflow and Pub/Sub

Apache Beam is the code, Dataflow is the engine that runs it, and Pub/Sub only delivers messages. Pub/Sub never computes. The same Beam pipeline handles batch and streaming.

If the answer could be a SQL query in BigQuery, it is not a Dataflow job. Use it for streaming, for logic SQL cannot express, or for data that must change before it lands: enriching a message in flight, stripping a card number, or filing a late event under its own timestamp.

## The streaming shape

```
events -> Pub/Sub -> Dataflow -> BigQuery
```

Pub/Sub decouples the sender. The source publishes and stops caring, and adding a fourth consumer next year changes nothing in the source.

Not run here: Pub/Sub end to end. The practice job read a replayed file from Cloud Storage instead.

## Do you need Dataflow at all?

| Need | Use |
| --- | --- |
| Pub/Sub straight into BigQuery, unchanged | BigQuery subscription |
| A light change to each message | BigQuery subscription with a Single Message Transform |
| Windowing, aggregating across messages, complex changes | Dataflow |
| No duplicates | Dataflow's Pub/Sub template, exactly-once by default. A BigQuery subscription is at-least-once, so plan to deduplicate downstream |

Not run here: BigQuery subscriptions.

## Templates

A template runs a pipeline with parameters only, so nobody writes Beam. The form holds pointers, and the code goes in a bucket, because the workers cannot read your laptop.

```bash
gcloud dataflow flex-template run my-job \
  --template-file-gcs-location gs://dataflow-templates/VERSION/flex/PubSub_to_BigQuery_Flex \
  --region REGION \
  --staging-location gs://my-bucket/staging \
  --parameters inputTopic=projects/PROJECT/topics/TOPIC,outputTableSpec=PROJECT:DATASET.TABLE
```

Use a dated version folder for `VERSION` in production, because `latest` can bring breaking changes. The table must already exist with a schema matching the messages. Failed rows go to `outputDeadletterTable`, which defaults to the output table's name plus `_error_records`.

Not run here: that command. What was run is the Cloud Storage text to BigQuery streaming template, through the console.

## What the practice job did

Create the destination table first, with a defined schema and partitioned on the timestamp column. Copy the schema file, a JavaScript function and the data to a bucket. Start the job from the template, query while rows arrive, then stop it.

A UDF (user-defined function) turns each raw line into a row. You give the form the `gs://` path and the function name, and Dataflow fetches it at run time.

The job graph is Beam's standard shape, read, transform, write, with a side path that sets failed rows aside instead of crashing the job. Check that side output. Rows can be failing there while the job looks healthy.

## Writing your own

Not run here. The shape, from the docs:

```python
(p
 | beam.io.ReadFromPubSub(topic=topic)
 | beam.Map(parse_message)
 | beam.io.WriteToBigQuery(table))
```

Read, transform, write is most Dataflow pipelines. Check the template list before writing one.

## Stopping a job

| | Drain | Cancel |
| --- | --- | --- |
| Reading new input | Stops soon after the request | Stops at once |
| Data in flight | Finishes and writes it | May be lost |
| Works on | Streaming only | Batch and streaming |
| Use it when | You want no data loss | The job is broken, or the data does not matter |

```bash
gcloud dataflow jobs list
gcloud dataflow jobs drain JOB_ID
gcloud dataflow jobs cancel JOB_ID
```

Draining can take a long time with a large backlog. Open windows close with partial results, so a restarted pipeline may fire the same window again.

## What breaks

**The job says Running with 0 workers.** "Startup of the worker pool ... failed to bring up any of the desired workers." There was no capacity for that machine type in the zone. Stop it, clone it under a new name, and set a different worker zone. The zone setting is under the optional parameters, not the regional endpoint dropdown.

**The form does not match the written steps.** Field names change. The `gs://` prefix is already filled in, so paste paths without it.

**The machine type will not change.** The default series was N1, and unticking the default box still showed N1 until the series was changed to E2.

**A preview shows no data.** Preview cannot see rows still streaming in. Run a `SELECT`.

## Watching it

| Question | Where |
| --- | --- |
| Which step is slow? | The job graph, which has numbers per step |
| Is the stream falling behind? | Data freshness and system latency in the job UI |
| What was the exact error? | Cloud Logging |
| Tell me when the backlog grows | A Cloud Monitoring alerting policy |

## Dead letters

A dead-letter topic is a second Pub/Sub topic for messages a subscription failed to deliver after a set number of tries, from 5 to 100. It only catches delivery failures. A message Dataflow received and your own checks rejected never failed in Pub/Sub, so write those to an error table inside the pipeline. Neither one tells a person. That is an alert.

## Bigtable as a destination

BigQuery answers "how many last quarter". Bigtable answers "what is this sensor's reading now": one known row key, millions of times a second. Use it when something needs millisecond lookups from the stream. Not run here.
