# BigQuery ML

Train and run models with SQL on data already in BigQuery. Nothing is exported and there is no separate service.

## The statements

| Statement | Answers |
| --- | --- |
| `CREATE MODEL` | Build it. Options name the model type and the label column |
| `ML.EVALUATE` | How good is it? |
| `ML.PREDICT` | Run it on new rows |
| `ML.CONFUSION_MATRIX` | The table of right and wrong predictions |
| `ML.ROC_CURVE` | Recall and the error counts at many thresholds |
| `ML.FORECAST` | Future values from a time-series model. Takes a horizon, not rows |

The label is the column you want to predict. Features are the columns used to predict it.

```sql
CREATE OR REPLACE MODEL my_dataset.purchase_model
OPTIONS (
  model_type = 'LOGISTIC_REG',
  input_label_cols = ['label']
) AS
SELECT
  IF(transactions IS NULL, 0, 1) AS label,
  device, country, pageviews
FROM my_dataset.sessions
WHERE visit_date < '2026-01-01';

SELECT * FROM ML.EVALUATE(MODEL my_dataset.purchase_model);

SELECT customer_id, predicted_label, predicted_label_probs
FROM ML.PREDICT(MODEL my_dataset.purchase_model,
  (SELECT customer_id, device, country, pageviews FROM my_dataset.sessions_today));
```

`predicted_label` is the class, and `predicted_label_probs` holds the probability behind it. For ranking, sort on the probability, not the class.

`IF` takes the condition, the value when true, then the value when false.

## The trap that made two results disagree

A training view was defined with `LIMIT 10000` and no `ORDER BY`. Each read could return different rows, so evaluation and the confusion matrix disagreed, at 85 and 146 real buyers, from "the same" view.

`LIMIT` without `ORDER BY` is a sample. If a view feeds a model, order it or save it as a table.

## Moving the threshold in SQL

Not run here: these calls. The practice run only read the default. The syntax comes from the docs. The default is 0.5, and this applies to binary classifiers only.

```sql
SELECT * FROM ML.EVALUATE(MODEL my_dataset.purchase_model,
  TABLE my_dataset.test_rows, STRUCT(0.02 AS threshold));

SELECT * FROM ML.CONFUSION_MATRIX(MODEL my_dataset.purchase_model,
  TABLE my_dataset.test_rows, STRUCT(0.02 AS threshold));

SELECT * FROM ML.PREDICT(MODEL my_dataset.purchase_model,
  TABLE my_dataset.new_rows, STRUCT(0.02 AS threshold));

-- Counts of each kind of error at many cut-offs
SELECT threshold, recall, true_positives, false_positives, false_negatives
FROM ML.ROC_CURVE(MODEL my_dataset.purchase_model, TABLE my_dataset.test_rows,
  GENERATE_ARRAY(0.01, 0.5, 0.01));
```

Moving the threshold does not retrain anything. See `judging-a-model.md` for choosing it.

## Options worth knowing

Not run here: these options, from the docs.

| Option | Does | Watch out for |
| --- | --- | --- |
| `DATA_SPLIT_METHOD` | `AUTO_SPLIT` by default. Also `RANDOM`, `CUSTOM`, `SEQ`, `NO_SPLIT` | Auto uses all rows for training under 500 rows. For rows ordered in time, use `SEQ` or `CUSTOM` with `DATA_SPLIT_COL`, so you test on the most recent period |
| `AUTO_CLASS_WEIGHTS` | Weights classes inversely to frequency | For logistic regression. Without it, the model may favor the common label |
| `ENABLE_GLOBAL_EXPLAIN` | Computes feature importance at training | Needed before `ML.GLOBAL_EXPLAIN` |

`ML.EVALUATE` with no table returns the metrics generated during training. Pass your own table to score held-out rows you chose.

Hyperparameters such as `LEARN_RATE` and `NUM_TREES` are per algorithm. The `OPTIONS` block only accepts what is valid for that `model_type`, and a wrong one errors.

Gradient boosted trees usually beat logistic regression on tabular business data, because they learn interactions such as "this matters only for new customers". Logistic regression stays popular because you can read and defend its weights.

## Forecasting

Not run here. `ARIMA_PLUS` learns from past values of one column over time. `ML.FORECAST` needs only a horizon:

```sql
SELECT *
FROM ML.FORECAST(MODEL my_dataset.sales_forecast,
  STRUCT(30 AS horizon, 0.8 AS confidence_level));
```

## Clustering

Not run here. `KMEANS` takes `num_clusters` and returns a cluster number per row. Naming the groups is your job. To shrink the number of columns instead of grouping rows, that is `PCA`, which is a different thing.

## Gemini from SQL

Not run here: this section comes from the docs. A remote model points at a model hosted elsewhere. It uses the same kind of connection as a Lakehouse table.

```sql
CREATE OR REPLACE MODEL my_dataset.gemini_model
  REMOTE WITH CONNECTION `LOCATION.CONNECTION_ID`
  OPTIONS (ENDPOINT = 'MODEL_NAME');

SELECT result, review
FROM AI.GENERATE_TEXT(
  MODEL my_dataset.gemini_model,
  (SELECT CONCAT('Three key words from this review: ', review) AS prompt, *
   FROM my_dataset.reviews
   LIMIT 5),
  STRUCT(0.2 AS temperature, 100 AS max_output_tokens));
```

- The input query must return a column named `prompt`.
- The connection's service account needs Agent Platform User (`roles/aiplatform.user`), in the right project. Without it, the call fails with a permission error.
- `ML.GENERATE_TEXT` is the older function, with different output column names. Use `AI.GENERATE_TEXT` for new work.
- Look up the current model name in the docs. Names change often.
- Every row is a model call, so test on a few rows first.

Remote models cannot be registered in Model Registry.
