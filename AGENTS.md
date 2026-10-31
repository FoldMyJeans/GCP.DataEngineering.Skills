# AGENTS.md

Operating rules for this repo, for any coding agent and any human editing it.

## What this repo is

Two things that share a subject:

- `skills/` holds agent skills for building on Google Cloud. To use one for a task, read its `SKILL.md`, then the reference it points to.
- `certifications/` holds study material, one folder per exam. It is written for people, and the skills do not depend on it.

## Content rules for the whole repo

The repo is public, so these apply to every file.

- Original work only. Sources are public Google Cloud documentation, public exam guides and hands-on practice.
- Nothing from a real exam, including from memory of one. No exam dumps.
- No video transcripts, copied course text or sample questions from someone else.
- No lab codes.
- No real company, employer, internal system or person's name, other than the author credit. Use "ABC Company" and placeholder hosts. No secrets, and no IP addresses that were only true for one run.

## Writing rules

- Plain engineer voice. Short direct sentences. Explain the why.
- Never use an em dash or an en dash. Use commas, periods, or parentheses.
- Never use a semicolon outside a fenced code block.
- Never state a count of things this repo contains. The number goes stale.
- Use a product's current name, with the old one in brackets the first time a file uses it: Knowledge Catalog (used to be Dataplex). Agents and older docs often know only the old name.

The working notes in `certifications/*/learning-notes` are deliberately long and unpolished. Keep their voice when adding to them.

## How the skills are structured

- One folder per skill under `skills/`, with a `SKILL.md` and a `references/` folder. The folder name matches the frontmatter `name`.
- `SKILL.md` frontmatter has `name` and `description`. The description is the trigger, so pack it with phrases and synonyms, under 1024 characters.
- Keep `SKILL.md` under 150 lines. Put the depth in references, one topic per file, each under 250 lines.
- Every skill folder is self contained. Never reference a path outside it. Point to another skill by name.
- Each fact has one owning file. Other files get a one or two line summary and a pointer.

## Show the evidence, and say what was not run

Skills here start from hands-on practice projects, not production systems, and the difference matters.

- Do not write that something is verified or battle tested. Give the detail only someone who hit the problem would know: the exact error, the wrong number.
- Label advice that was not run hands-on with "Not run here:". The reader should check it against the docs.
- If a claim could not be tested and the docs do not settle it, leave it out or say to check. Do not guess.
- When real projects add experience, add it and remove the label.

## Changing the repo

Work on a branch and open a pull request. Do not commit straight to main. When a skill changes, bump the `version` in `.claude-plugin/plugin.json` so installed copies pick it up.

Before committing, check by hand that relative links resolve, the rules above hold, and any JSON parses.

## Keep it local

Do not put this repo in a OneDrive synced folder. Git plus OneDrive can corrupt the repo.
