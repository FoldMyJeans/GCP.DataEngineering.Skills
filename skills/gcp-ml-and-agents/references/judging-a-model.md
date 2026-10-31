# Judging a model

The model outputs a probability. The threshold turns it into a decision. The action is somebody's job. Only the first is machine learning.

## The confusion matrix

Say which class is "positive" first. The four cells mean nothing until you do.

Read the name backward. The second word (Positive or Negative) is what the model said. The first word (True or False) is whether it was right. So a miss is always a False Negative.

| | Actually positive | Actually negative |
| --- | --- | --- |
| Predicted positive | True positive | False positive |
| Predicted negative | False negative | True negative |

The numbers come from the test set, rows held back during training. Written descriptions get this wrong: one set of written steps described its own 13 percent backward. Read the matrix, not the sentence.

## The metrics

| Metric | Question | Formula |
| --- | --- | --- |
| Recall | Of all real positives, how many did I catch? | TP / (TP + FN) |
| Precision | Of everything I flagged, how many were right? | TP / (TP + FP) |
| AUC | Given one positive and one negative, how often is the positive scored higher? | Area under the ROC curve |

"Catch them all" is recall, because you fear a miss. "Only flag the real ones" is precision, because you fear a false alarm. You cannot maximize both.

AUC ignores the threshold. That is why an AUC of 0.98 can sit next to a recall of 10 percent, and both are true. With a low base rate, a well-calibrated model rarely says above 0.5 for anyone.

## Picking the threshold

Nothing about the model changes when the threshold moves. A person scored 0.35 is still 0.35. The counts move because people cross a line you moved. A new model takes hours, and a new threshold takes seconds.

Choose by cost, not by feel:

```
total cost = (false negatives x cost of a miss) + (false positives x cost of a false alarm)
```

Illustrative loan case per 100 applicants (80 repay, 20 default), where a default costs 10,000 and refusing a good customer costs 1,500:

| Threshold | Missed defaults | Refused good customers | Total cost |
| --- | --- | --- | --- |
| 0.5 | 13 percent | 0 percent | 26,000 |
| 0.3 | 5 percent | 4 percent | 14,800 |
| 0.2 | 2 percent | 12 percent | 18,400 |

Cost falls, bottoms out and rises. That U shape is what you look for.

To compare two models, compare each at its own best threshold, on total cost. A model can make more errors and still win by making cheaper ones.

**Who decides.** An engineer can say "at 0.2 you catch 80 percent and waste 300 escalations". Whether 300 is acceptable is the product owner's call. If nobody makes the call, the model ships at 0.5.

## Reading the charts

| Chart | Answers |
| --- | --- |
| Precision-recall curve | How good is the trade-off overall? |
| ROC curve | Is it good at ranking? Hugging the top left is good, and the diagonal is a coin flip |
| Precision-recall by threshold | Which threshold? The only one with the threshold on an axis |

The chart narrows the range and rules out bad regions. Choosing within it takes the cost numbers, which are not in the chart.

## Traps

**Data leakage.** Using a fact you only know afterward. Predicting whether a ticket breaches its SLA with `resolution_hours` as a feature looks 99 percent accurate in training, and the column is blank when the ticket just opened. On every feature, ask what is actually populated at the moment you press predict.

**Overfitting.** The model memorized the rows. The tell is the gap:

| | Training | Test |
| --- | --- | --- |
| Healthy | 82 percent | 79 percent |
| Overfit | 99 percent | 61 percent |

A single impressive number quoted alone is usually the training score. Ask for both.

**No baseline.** Score the dumbest rule the same way: always the average, or always the common class. If the model barely beats it, it is not useful yet.

**A random split on time data.** That trains on December and tests on November, predicting the past from the future. Train on the earlier period and test on the latest. Scores drop, and the drop is real.

**Training-serving skew.** The training SQL and the live path compute a feature slightly differently, for example strictly before versus including the same minute. No error appears, offline metrics stay good, and live predictions are subtly wrong. Define the feature once.

## Feature importance

A bar chart of how much each feature contributed. Ask two things:

1. Is anything suspiciously dominant? That is usually leakage. `ticket_id` at the top means something is badly wrong.
2. Is anything near zero? Drop it next time.

Importance is not causation. Two correlated features split the credit and both look moderate. Domain knowledge beats technical knowledge when reading it.

| Tool | Scope | Answers |
| --- | --- | --- |
| Feature importance | Whole model | What does it rely on? |
| Feature attributions | One prediction | Why this decision? |
| Example-based explanations | One prediction | Which training rows are similar? |

Global is about the model. Local is about a decision, and that is what goes in a letter to a declined applicant. Attributions add latency and cost to every request. Not run here: attributions on a live endpoint.

Check for protected characteristics. A feature like age at the top of a lending model is a problem in most jurisdictions, and catching it before a regulator does is the point.

## Feature engineering

A raw timestamp is just a big number. Features are what the model can use:

| Raw | Feature | Why |
| --- | --- | --- |
| `opened_at` | `opened_hour`, `is_weekend` | Now it can learn "2am tickets breach more" |
| `caller_email` | `caller_domain` | The person is noise, and the company is the pattern |
| `opened_at`, `resolved_at` | `resolution_hours` | The gap is the signal. Check it for leakage |
| Past tickets from this caller | `caller_breach_rate_90d` | History the current row does not contain |

The last kind is usually where the predictive power lives. Compute it only from rows before this one (`closed_at < opened_at`), which keeps it out of leakage.

Scale inputs for neural networks, so a feature in the thousands does not drown out one between 0 and 1.
