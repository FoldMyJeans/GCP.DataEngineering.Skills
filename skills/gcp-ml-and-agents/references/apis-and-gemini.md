# Pre-trained APIs and Gemini

The model never comes to you. Your text goes to Google's server, the model runs there, and JSON comes back. The same call works from a laptop or a VM.

## Which one

| The job | Use | Why |
| --- | --- | --- |
| One fixed question at volume, needing a number every time | Specialized API | Same structure every call, cheaper, faster |
| Open-ended or mixed questions, summaries, drafting | Gemini | Flexible |
| Labels specific to your business, such as "is this ticket a false alarm" | BigQuery ML or AutoML | A general model knows nothing about your domain |

A score of `-0.8` can be stored in a column, averaged by month, thresholded and charted. You cannot `AVG()` a paragraph. That is the case for the older single-purpose APIs in one line. Not run here: comparing the cost of the two at volume.

## The Natural Language API

```bash
curl "https://language.googleapis.com/v1/documents:analyzeEntitySentiment?key=${API_KEY}" \
  -s -X POST -H "Content-Type: application/json" \
  --data-binary @request.json > result.json
```

```json
{
  "document": { "type": "PLAIN_TEXT", "content": "I liked the food but the service was terrible." },
  "encodingType": "UTF8"
}
```

The method name in the URL is the only thing that changes between kinds of analysis. To read a file in a bucket, use `gcsContentUri` in place of `content`.

| Method | What it breaks the text into | In v2? |
| --- | --- | --- |
| `analyzeEntities` | Things: people, places, organizations | Yes |
| `analyzeSentiment` | Sentences, each with a score | Yes |
| `classifyText` | One topic for the whole document | Yes |
| `analyzeEntitySentiment` | Things, each with its own score | No, v1 only |
| `analyzeSyntax` | Single words and their grammar | No, v1 only |

The practice calls used v1. The current reference lists v2 without the last two methods, so check the version before building on them.

### Reading the answer

- **Salience** runs from 0 to 1, for how central an entity is. In one sentence a person scored 0.798 and the book series they wrote scored 0.015. It separates what a text mentions from what it is about.
- **Score and magnitude** are different. Score runs negative to positive. Magnitude is the total emotion in either direction. A half glowing, half furious review scores near zero, the same as a flat one, and magnitude tells them apart.
- **Entity sentiment is the one that earns its keep.** "I liked the food but the service was terrible" averages to neutral as a whole. Per entity, it says the service is the problem.
- **An entity is a thing, not a string.** The same country named in two languages comes back with the same Knowledge Graph ID, so reviews in several languages can be grouped without translating. The language is detected if you do not state it.
- **`UNKNOWN` fields do not mean failure.** The response format is shared across languages, and some grammar features do not exist in English.

### What breaks

- **The model is not always right.** A nationality came back as a location, and a job title came back as a person.
- **`export API_KEY=...` prints nothing.** Check it with `echo $API_KEY`.
- **SSH in the browser failed** with a "Cloud Identity-Aware Proxy" not authorized error (code 4033). The dialog's retry without the proxy connected. `gcloud compute ssh` from Cloud Shell is the fallback.
- **The reply differs from written examples.** Trust the output.

An API key proves you may call and says who gets billed. Restrict it to the one API, and do not put it in code or a public repo.

## Calling Gemini from Python

Not run here: this code, which comes from the SDK docs. Use `google-genai`. The older `google-generativeai` package is deprecated, and older examples with `genai.configure` and `GenerativeModel` use it.

```bash
pip install google-genai
```

```python
from google import genai
from google.genai import types

client = genai.Client(enterprise=True, project="PROJECT_ID", location="us-central1")

response = client.models.generate_content(
    model="MODEL_NAME",
    contents="Classify this ticket: printer offline on the third floor",
    config=types.GenerateContentConfig(
        temperature=0,
        system_instruction="Reply with one word: hardware, software or access.",
    ),
)
print(response.text)
```

`enterprise=True` is how the current docs reach Agent Platform. Older samples say `vertexai=True`, so check the SDK docs for your version. Look up the model name too, because names change often and an agent's memory of them goes stale.

Ask for a fixed shape when code will read the answer: set `response_mime_type` to `application/json` and pass a schema in `response_json_schema`.

## Settings

| Term | When | Examples |
| --- | --- | --- |
| Parameters | Learned during training | Weights |
| Hyperparameters | Set before training | `learn_rate`, `num_trees`, `max_depth` |
| Inference parameters | Set per request, generative only | Temperature, top-K, top-P, max output tokens |

Temperature 0 suits extraction and classification. Higher suits creative work. The console can label these "model parameters", which confuses people. Some models show a "Thinking level" setting instead of Temperature and Top-P, so check what your model exposes.

Choose a faster, cheaper model first, and move up only when the answers fall short. Model Garden is the catalog to pick from. It is a shelf, not a runtime.

To run Gemini over rows in a table, see `bigquery-ml.md`.
