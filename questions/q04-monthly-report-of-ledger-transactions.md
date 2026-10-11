---
question: 4
title: "Monthly-Report-Of-Ledger-Transactions"
framework: "T-A-G (Task-Action-Goal)"
provider: "Anthropic"
model: "Claude Opus 5.5"
effort: "default"
status: "final"
---

## Scenario

Jennifer Parker is finishing a presentation for Goldie Wilson on transaction
growth by category over the last 6 months. She needs consolidated numbers but
doesn't write SQL. The data lives in Ledger (PostgreSQL), across two tables
(see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
TASK: Write a PostgreSQL query for Ledger that consolidates completed
transactions per month and category over the last 6 rolling months.
Schema:
CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);

CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
Categories in production: subscription, one_time, refund, credit_adjustment.

ACTION:
- Today is 2026-04-24. Window: 2025-10-24 (inclusive) to 2026-04-24 (inclusive).
- Filter status = 'completed'.
- Group by month (TO_CHAR 'YYYY-MM') and category.
- Per row: transaction_count and total_volume_brl = SUM(amount_cents)/100,
  rounded to 2 decimals.
- Order by month ASC, then category ASC.

GOAL: Jennifer (PM, doesn't write SQL) will present transaction growth by
category to the CEO next week, so the numbers must be consolidated and
ready to paste into a slide. Return only the SQL, then list any assumptions
you made (e.g. which date column defines the month).
```

## Model

**Model used:** Claude Opus 5.5 (Anthropic)

**Why this model:** the task depends on getting date boundaries, time zone,
and cent-to-real conversion exactly right, and the model reliably surfaces
the assumptions it makes when the prompt asks for them.

## Output

```sql
SELECT
    TO_CHAR(DATE_TRUNC('month', created_at AT TIME ZONE 'America/Sao_Paulo'), 'YYYY-MM') AS month,
    category,
    COUNT(*)                            AS transaction_count,
    ROUND(SUM(amount_cents) / 100.0, 2) AS total_volume_brl
FROM transactions
WHERE status = 'completed'
  AND created_at >= TIMESTAMPTZ '2025-10-24 00:00:00-03'
  AND created_at <  TIMESTAMPTZ '2026-04-25 00:00:00-03'
GROUP BY 1, 2
ORDER BY month ASC, category ASC;
```

Assumptions:
1. `created_at` defines the month. It's indexed and marks when the
   transaction happened. If Finance wants the settlement date instead, swap
   both the filter and the grouping to `completed_at`.
2. Dates are interpreted in `America/Sao_Paulo` (UTC-3, no DST since 2019).
3. Both 2025-10-24 and 2026-04-24 are included, so the upper bound is
   `< 2026-04-25`. October 2025 and April 2026 are therefore partial months,
   which is worth noting on the slide.
4. `refund` and `credit_adjustment` are summed as stored. If refunds are
   stored as positive values, the total volume will be overstated, so
   confirm the sign convention.
5. A month/category pair with no completed transactions won't appear as a
   zero row.

## Rationale

**Task** states what to build (a monthly per-category consolidation) and
carries the full schema plus the production categories, so the model works
from real columns and can reason about `refund`/`credit_adjustment` (see
assumption 4). **Action** lists every constraint from the brief as a
checklist, and each one maps to a line of the query: the explicit window
anchored on "today is 2026-04-24", `status = 'completed'`, the `YYYY-MM`
grouping, the two metrics with `/ 100.0` and `ROUND(..., 2)`, and the
ascending sort. **Goal** gives the purpose (Jennifer's growth presentation to
the CEO), which explains why the output must be slide-ready and why the
model flags partial months, plus the instruction to return only SQL and list
assumptions, which surfaced the date-column and time-zone decisions.
