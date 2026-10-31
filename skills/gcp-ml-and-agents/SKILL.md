---
name: gcp-ml-and-agents
description: >-
  Choose and build ML and gen AI on Google Cloud: pre-trained APIs, Gemini, BigQuery ML, AutoML and
  custom training in Agent Platform (used to be Vertex AI), plus Agent Studio and the Agent
  Development Kit (ADK). Covers CREATE MODEL, ML.EVALUATE, ML.PREDICT, thresholds, precision and
  recall, AUC, data leakage, feature engineering, endpoints vs batch, Pipelines, drift, RAG vs
  fine-tuning, and when a job deserves an agent. Use whenever the user wants to predict, classify,
  forecast, cluster or score something, call Gemini or the Natural Language API, read a model's
  metrics, pick a cut-off, deploy a model, or build an agent. Trigger on "train a model", "BigQuery
  ML", "AutoML", "Vertex AI", "my model has low recall", "confusion matrix", "threshold", "feature
  engineering", "leakage", "deploy to an endpoint", "sentiment analysis", "ML.GENERATE_TEXT", "RAG",
  "fine-tune", "ADK", "build an agent".
---

# GCP ML and agents

This skill covers choosing a model, training it, judging it, serving it, and building agents. Preparing the data in SQL is `gcp-bigquery`. Scheduling a scoring job is `gcp-pipelines-and-orchestration`.

It starts from hands-on practice projects. Advice that was not run hands-on is labelled "Not run here:", so check it against the current docs before relying on it.

## Start at the top of the ladder

Stop at the first rung that does the job. Each step down buys control with effort.

| Rung | Use when | You bring |
| --- | --- | --- |
| Pre-trained API or Gemini | A common task, or open-ended text work, with no labels | Nothing |
| BigQuery ML | Labeled data already in BigQuery, a team that writes SQL | SQL |
| AutoML in Agent Platform (used to be Vertex AI) | Your own labels, tabular or image, minimal code | Clicks |
| Custom training | You need control that nothing above can express | Code |

Three questions pick the rung: do you have labels, what is the data type, and does the team write SQL or code? If the answer is in the pixels, metadata in SQL will not find it, so use an image model.

| The ask | Type | In BigQuery ML |
| --- | --- | --- |
| Which category, yes or no | Classification | `LOGISTIC_REG`, `BOOSTED_TREE_CLASSIFIER` |
| What number | Regression | `LINEAR_REG`, `BOOSTED_TREE_REGRESSOR` |
| Future values over time | Forecast | `ARIMA_PLUS`, not `LINEAR_REG` |
| Group rows, no labels | Clustering | `KMEANS`. You name the groups yourself |
| Generate or summarize text | Gemini | Remote model |

Logistic regression is classification, despite the name.

## Is this even an agent?

**If you can draw the flowchart, write the code.** An agent earns its place only when the steps vary by input and cannot be known in advance. A Word to PDF converter is a library call. Triage that decides which system to check next, then re-decides based on what came back, is an agent.

## Fine-tuning or RAG

**Fine-tuning teaches format and behavior. RAG supplies facts.** A fine-tuned model produces confidently wrong answers in your house style. Facts change daily, and re-indexing daily is routine while retraining daily is absurd. Try prompt design first, because it costs nothing.

## Rules of thumb

- **Ask whether it is a ranking tool or a decision tool.** Ask this before asking if the model is good enough. "Who are the 1,000 most likely to buy" only needs a good order, so a high AUC is enough. "Approve this, yes or no" needs a cut-off.
- **A high AUC with low recall is not a bug.** One model scored AUC 0.98 and recall about 10 percent, with about 1.5 percent of visitors buying. Almost nobody got a probability above the default 0.5. That is the threshold, not the model.
- **0.5 is not neutral, it is unexamined.** The threshold is a business choice about what each mistake costs, and it is the cheapest lever you have. See `references/judging-a-model.md`.
- **A near perfect score is a warning.** Look for leakage. At the moment you press predict, do you actually have this column? If it only exists after the event, drop it.
- **Beat the baseline.** 99.5 percent accuracy on data where 99.5 percent of rows are negative is what "always say no" scores.
- **Do not deploy out of habit.** Batch prediction needs no endpoint, and the machine behind an endpoint runs from deploy until undeploy, whether or not anyone calls it.
- **Exclude ID columns.** An arbitrary label cannot cause anything, and a model may find fake patterns in it.
- **Temperature is not training.** It is set per request, on generative models only, and changes nothing about the model.

## Before you hand this back

- The score came from rows the model never saw, split by date if the rows are ordered in time.
- It was compared to a baseline, not quoted alone.
- Every feature was checked for leakage.
- Someone with business context chose the threshold, or the threshold is flagged as still the default.
- Anything deployed has an owner and a plan to undeploy it.
- Regulated decisions use a model you can explain, and protected characteristics were checked.

## References in this skill

- `references/bigquery-ml.md`: `CREATE MODEL`, evaluate, predict, thresholds in SQL, data splits, and remote Gemini models.
- `references/judging-a-model.md`: confusion matrix, precision and recall, picking a threshold by cost, leakage, overfitting, and feature importance.
- `references/automl-and-serving.md`: AutoML tabular, budgets, endpoints vs batch, monitoring, Pipelines, and features.
- `references/apis-and-gemini.md`: pre-trained APIs, the Natural Language API, calling Gemini, and when each wins.
- `references/agents.md`: Agent Studio vs ADK, tools, the loop, prompts, grounding, and deployment.
