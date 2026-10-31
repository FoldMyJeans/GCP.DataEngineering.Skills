# GCP.DataEngineering.Skills

Data engineering on Google Cloud, in one place. Free study material for the data certifications, and agent skills for building real projects with what those certifications teach.

Unofficial. Not affiliated with or endorsed by Google.

## What are you here for?

| You want to | Go to |
|---|---|
| Pass the Associate Data Practitioner exam | [certifications/associate-data-practitioner](certifications/associate-data-practitioner/README.md): the handbook, practice sets, hands-on notes and a two week plan |
| Pass the Professional Data Engineer exam | [certifications/professional-data-engineer](certifications/professional-data-engineer/README.md): not written yet |
| Build something on Google Cloud with an AI agent | [The skills](#the-skills) below |

You only need the part you came for. Nothing in one depends on the other.

## The skills

Exam notes teach you to pick the right answer. A skill tells an agent how to build the thing, and where it goes wrong. These come from working through Google Cloud hands-on, so they carry the exact errors and traps I hit. Where a skill goes beyond what I ran myself, it says so.

| Skill | What it does | Example trigger |
| --- | --- | --- |
| `gcp-build-playbook` | Plan a whole project end to end and route to the right skill: phase order, decisions that are expensive to reverse, choosing storage, and working with a console that no longer matches written steps. | "where do I start", "which database should I use", "plan this build" |
| `gcp-ingestion-and-migration` | Choose how data gets into Google Cloud, then set it up: file transfers, `bq load`, Datastream replication, and querying files without loading them. | "load this CSV into BigQuery", "replicate Postgres to BigQuery", "move this data from S3" |
| `gcp-bigquery` | Write SQL that is actually right, design tables so queries stay cheap, and control and recover the data. | "my numbers don't match", "why did this query cost so much", "hide this column" |
| `gcp-pipelines-and-orchestration` | Choose and build transforms and what runs them: Dataform, Dataflow, Pub/Sub, serverless Spark, Airflow, Workflows and Eventarc. | "build a pipeline", "stream this into BigQuery", "run when a file lands" |
| `gcp-ml-and-agents` | Choose a model, judge it, serve it, and build agents: BigQuery ML, AutoML, Gemini, and the Agent Development Kit. | "train a model", "my model has low recall", "build an agent" |

## Use the skills

The skills follow the Agent Skills standard: a `SKILL.md` with `name` and `description` frontmatter, plus a `references/` folder. It is an open format, and each skill folder is self contained.

### Claude Code

```
claude plugin marketplace add FoldMyJeans/GCP.DataEngineering.Skills
claude plugin install gcp-data-engineering-skills@gcp-data-engineering-skills
```

Run those from a project folder, then restart your session so the skills load. Update later with `claude plugin update gcp-data-engineering-skills`.

### Codex, Cursor, and other agents that read ~/.agents/skills

PowerShell (Windows):

```
git clone https://github.com/FoldMyJeans/GCP.DataEngineering.Skills
Copy-Item -Recurse -Force GCP.DataEngineering.Skills/skills/* ~/.agents/skills/
```

macOS or Linux:

```
git clone https://github.com/FoldMyJeans/GCP.DataEngineering.Skills
mkdir -p ~/.agents/skills && cp -R GCP.DataEngineering.Skills/skills/* ~/.agents/skills/
```

### Any other AI tool

A skill is a folder of markdown. Give the tool the repo link and tell it which skill to read, for example `skills/gcp-ingestion-and-migration/SKILL.md`. If it cannot browse, paste the `SKILL.md` in, then the file under `references/` it points you to.

## Good to know

- Everything here is original: my own notes, questions and skills, put together from public Google Cloud documentation, public exam guides and my own hands-on practice. Nothing comes from an actual exam.
- Google renamed several products in 2024 to 2026. This repo uses the new name and puts the old one in brackets, for example Managed Service for Apache Spark (used to be Dataproc).
- These can be wrong in places. Where they disagree with the Google Cloud docs, trust the docs. Found a mistake? Please open an issue.

## Contributing

See [AGENTS.md](AGENTS.md) for the writing rules and how the skills are structured.

Created by [Reza Seifollahi](https://www.linkedin.com/in/mrezasf/), 2026.
