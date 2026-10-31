# Working in the console

The console changes faster than any written steps, including these. These are the patterns that held across many runs.

## When the steps do not match the screen

Assume written steps are a little stale. Look for the modern equivalent of a missing control, rather than hunting for the exact name.

| What happened | What worked |
| --- | --- |
| The "+ Add data" button was not where the steps showed it | The Explorer tree: Connections, three-dot menu, Create connection. Or the same thing from Cloud Shell |
| A checkbox was now a minus-circle icon at the end of the row | Read the row, not the instructions |
| A form's field names differed from the steps | Match by meaning. Paths may already have the `gs://` prefix |
| A setting the steps named did not exist on that model | Check what the model actually offers |
| Sample output in the steps differed from the real output | Trust the output. Sample files get updated |
| Steps described their own chart backward | Read the chart, not the sentence |

**A `gcloud` or `bq` command usually does the same job as a missing button**, and its output is a more reliable record than a page. The IAM page caches: a role just granted did not show until a refresh.

**Use a command for anything you will do twice.** Clicks leave nothing to review or repeat.

## Search bars land on the wrong product

Searching a product name in the console's main search bar can open the wrong page. It happened twice: once for the Agent Platform (used to be Vertex AI) product page instead of a connection type, and once for Data Studio Pro instead of Data Studio. Search inside the tool you are working in.

## Stuck or slow

- **When a Create button seems stuck, check the list before submitting again.** A dataset seemed to hang for about two minutes and had in fact been created.
- **Start slow resources first.** A Cloud SQL instance can take 5 to 10 minutes to build, so start it and do other work meanwhile.
- **Long training does not need watching.** AutoML took about an hour on a small dataset.

## Terminals

- **Cloud Shell** is a small free temporary machine with 5 GB of home disk. Treat it as a scratch desk. Real files go in a bucket.
- **An SSH window** belongs to a VM. Check which one you are typing in.
- **Environment variables** live only in the session where you set them. A new window loses them. `export` prints nothing, so check with `echo`.
- **A password prompt that fails at once** can mean leftover line breaks from a multi-line paste were read as an empty password. Press Enter a few times and retry.
- **SSH in the browser can fail** on Identity-Aware Proxy (code 4033). The dialog's retry without the proxy worked, and `gcloud compute ssh` from Cloud Shell is the fallback.
- **`bq` is only a client.** The table lives in BigQuery and shows up just the same in the console.

## Signing in

- Check the avatar before connecting any data, so you do not attach the wrong account.
- Close every incognito window between sessions with different accounts. A leftover account from an earlier one broke the next sign-in.

## Moving from a practice project to a real one

Practice shortcuts that must not carry over:

| Shortcut | Real version |
| --- | --- |
| A superuser or your own login for a job | A dedicated service account or database user with only what it needs |
| An unencrypted database connection | SSL |
| IP allowlisting of a public database | Usually private connectivity. Not run here |
| Roles granted across the whole project | Roles on the dataset or bucket |
| A large machine picked for speed | The smallest that does the job |
| Sandbox tables that expire after 60 days | Billing enabled and expiration set on purpose |
| Writing directly in the web editor | Git with review |

Staying inside the plan matters in a practice project: bigger machines or extra products can get the project shut down. In a real project, the same restraint keeps the bill down.

## When time is limited

Do the main steps first and explore afterward. One exercise was lost to the clock by doing it the other way around.

The optional steps are often the real lesson. Steps that are not scored, such as the query that proves a column is locked, are easy to skip and are the reason for doing the work.
