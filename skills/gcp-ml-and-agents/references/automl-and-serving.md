# AutoML and serving

Agent Platform (used to be Vertex AI) holds the whole workflow: prepare data, train, evaluate, deploy, predict, monitor. It is a loop. Training shows you need better features, and monitoring shows when to retrain.

## AutoML for tabular data

You point it at labeled data. It handles encoding, searches model designs and settings, and ensembles the best ones.

| Setting | What to know |
| --- | --- |
| Rows | At least 1,000, which the docs say may still not be enough for a good model |
| Objective | Regression, classification or forecasting |
| Target | The column to predict |
| ID columns | Exclude them. The control was a minus-circle icon at the far right of the row, not the checkbox the written steps described |
| Budget | Node hours. One is one machine for one hour. It is a cap on the search, not a duration |
| Early stopping | Ends training when scores stop improving. Leave it on |
| Split | Random 80/10/10 by default. Use chronological when rows are ordered in time |

What you get: metrics, feature importance and a deployable model. What you do not get: the architecture or the settings. Where you must explain each decision, as in regulated lending, a simple model in BigQuery ML or custom training is easier to defend.

## What breaks

**Training takes longer than you have.** A small dataset of 2,050 rows with 3 features took about an hour. Plan around it.

**The Create button hangs.** Creating an empty dataset seemed stuck for about two minutes, and the dataset had in fact been created. Check the Datasets list before submitting again.

**The "total feature columns" counter is off by one.** It includes the target.

## Model and endpoint

The model is the artifact in Model Registry. A deployed model is that artifact running behind an endpoint. One model can be deployed more than once, which is how traffic is split between versions. Containers and packages go in Artifact Registry, and models go in Model Registry.

| Way to serve | How | Use when |
| --- | --- | --- |
| Online | Synchronous, to a model deployed on an endpoint | Something is blocked waiting |
| Batch | Asynchronous, sent to the model with no endpoint | Nobody is waiting |
| Edge | The model is copied to a device | No internet, or the round trip is too slow |

**The deciding question: is anything blocked waiting for this answer?** A risk-ranked work queue at 8am is as useful as one computed in 70 ms, and it needs no endpoint. Online earns its cost when a decision is truly blocked, such as declining a card.

The machine behind an endpoint is what costs money, and it runs from deploy until undeploy, whether or not anyone calls it. Undeploying is a step of its own.

A prediction returns raw probabilities per class, not a yes or no:

```json
"scores": [0.99, 0.01],
"classes": ["0", "1"]
```

Applying a threshold is your job. See `judging-a-model.md`.

## Drift and monitoring

The world changes and the model does not. One deployed model carried a training date several years old in its name, still answering.

Not run here: Model Monitoring. From the docs:

| Term | Means |
| --- | --- |
| Training-serving skew | Live feature distribution differs from training |
| Inference drift | Live feature distribution changes over time |

It compares distributions against a baseline. It does not measure accuracy, because it never learns whether the loan was repaid. It can tell you conditions changed, not that the model got worse. To measure accuracy, join predictions to real outcomes later.

## Pipelines

Not run here: this section. Agent Platform Pipelines automates the workflow, serverlessly, with Kubeflow Pipelines or TensorFlow Extended (TFX).

Automate only after you have run the workflow by hand once. Manual comes first, and it is not a failure.

A component is one step that declares its inputs, outputs and container. Because those are declared, the orchestrator can run steps in parallel, skip a step whose inputs did not change, retry a failure, and track which dataset and run produced each model.

```python
@pipeline()
def training_pipeline():
    dataset = TabularDatasetCreateOp(bq_source="bq://...")
    model = AutoMLTabularTrainingJobRunOp(dataset=dataset.output)
    check = eval_metrics(model=model.output, threshold=0.85)
    with dsl.If(check.outputs["deploy"] == "true"):
        endpoint = EndpointCreateOp()
        ModelDeployOp(model=model.output, endpoint=endpoint.output)
```

Running that Python trains nothing. It builds a definition, which you compile and submit. That one `If` is what makes it MLOps: the pipeline decides whether the model is good enough. Check the prebuilt components before writing your own. Keep each one to a single job, so it can be reused.

The reason to write a pipeline instead of clicking is continuous training, because in ML the data changes and a correct system becomes wrong with nobody touching it.

## Feature Store

Not run here. A central place for feature definitions and values. Building `avg_ticket_age_30d` once means a later model reuses the definition instead of rewriting the SQL and getting a slightly different answer. Offline serves training and batch from BigQuery. Online serves one row at low latency, and needs a store kept fresh. Do not build the online path unless something is blocked waiting.

## Custom training

Not run here. Your code in TensorFlow, PyTorch or scikit-learn, run as a training job in a prebuilt or custom container, from Workbench or Colab Enterprise. Reach for it when you have reason to believe your structure beats what a search would find.
