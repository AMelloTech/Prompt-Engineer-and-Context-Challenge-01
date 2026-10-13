# Prompt Engineering and Context — Challenge 01

Postgraduate challenge: 8 questions applying prompt engineering frameworks to
fictional scenarios at **Hill Valley Tech**. See the full company and team
context in [`context/hill-valley-tech.md`](context/hill-valley-tech.md).

## Deliverable

Each question lives in its own Markdown file under [`questions/`](questions/),
with 4 required fields: **Prompt**, **Model** (+ reason for the choice),
**Output**, and **Rationale** (mapping the framework's components to the
prompt). Question 08 compares the chosen framework against 2 alternatives.

At least 2 distinct providers (OpenAI, Anthropic, Google, Meta, or local via
Ollama) are used across the challenge. Bad outputs are recorded too, with a
note on what would be done differently.

## Repository structure

```
.
├── README.md                      this file — overview and index
├── context/
│   └── hill-valley-tech.md        company and team scenario, referenced by every question
└── questions/
    ├── _template.md                template followed by every question
    ├── q01-*.md
    ├── ...
    └── q08-*.md
```

**Why this structure:** one file per question keeps the git history
granular — each commit shows the evolution of a specific prompt — and the
front matter (`framework`, `provider`, `model`, `effort`) works as
filterable metadata without repeating the company context in every file.
This choice will be revisited in Chapter 4 (Prompt Creation and
Versioning).

## Question index

| # | Framework | Provider | Model | Status |
|---|---|---|---|---|
| [01](questions/q01-docker-para-o-lift.md) | R-T-F | Anthropic | Claude Sonnet 5 | final |
| [02](questions/q02-backup-script-of-ledger.md) | R-T-F | OpenAI | GPT 6.0 Luna | final |
| [03](questions/q03-report-of-cost-reduction-for-cloud.md) | T-A-G | Anthropic | Claude Opus 5 | final |
| [04](questions/q04-monthly-report-of-ledger-transactions.md) | T-A-G | Anthropic | Claude Opus 5.5 | final |
| [05](questions/q05-legacy-deploy-modernization.md) | B-A-B | OpenAI | GPT 6 Luna | final |
| [06](questions/q06-terraform-s3-module-internal-standard.md) | C-A-R-E | Anthropic | Claude Opus 5.5 | final |
| [07](questions/q07-runbook-for-recurring-alert.md) | R-I-S-E | Anthropic | Claude Opus 5.5 | final |
| [08](questions/q08-placeholder.md) | — (compared against 2 alternatives) | — | — | pending |
