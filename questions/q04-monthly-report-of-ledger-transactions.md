---
question: 4
title: "Monthly-Report-Of-Ledger-Transactions"
framework: "T-A-G (Task-Action-Goal)"
provider: "OpenAI"
model: "GPT 6.0 Luna"
effort: "high"
status: "final"
---

## Scenario

The team needs the last 6 months of transactions grouped by category. The
data lives in Ledger (PostgreSQL), split across two tables (see the general
context in [`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
TASK: We need to delivery the informations regarding the last 6 months of transactions per category. The information is inside 2 tables in PostgreSQL. CREATE TABLE transactions (
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
ACTION: Use the table information to create a SQL query that can give us the last 6 month of transactions.
GOAL: We only want in the report transaction that have status as completed, the amount_cents field is in cents and it needs to present the output in REAIS with 2 decimals.The starting date is 2026-04-24 grouped by month in this format and category. with 2 metrics per line: Transaction count and total volume in reais. Order by month crecent, and category also crescent
```

## Model

**Model used:** GPT 6.0 Luna (OpenAI), high effort

**Why this model:** keeps the provider mix balanced (OpenAI vs. Anthropic),
and high effort suits a task where date boundaries, cent-to-real conversion,
and grouping all have to be exactly right.

## Output

Assuming 2026-04-24 is the inclusive start date and the report runs through
the current time, use:

```sql
SELECT
    TO_CHAR(DATE_TRUNC('month', created_at), 'YYYY-MM') AS month,
    category,
    COUNT(*) AS transaction_count,
    ROUND(SUM(amount_cents)::numeric / 100, 2) AS total_volume_reais
FROM transactions
WHERE status = 'completed'
  AND created_at >= TIMESTAMPTZ '2026-04-24 00:00:00'
  AND created_at <= NOW()
GROUP BY DATE_TRUNC('month', created_at), category
ORDER BY DATE_TRUNC('month', created_at) ASC, category ASC;
```

The month is formatted as YYYY-MM. Since only transaction fields are needed,
the query does not join customers.

## Rationale

**Task** frames the business need (last 6 months of transactions per
category) and carries the full schema of both tables, so the model worked
from real column names and types instead of guessing. **Action** asks for a
SQL query over that schema, and the model concluded that `customers` wasn't
needed and skipped the join, saying so explicitly. **Goal** holds the exact
constraints, and each one maps to a line of the query: `status = 'completed'`,
the `/ 100` with `ROUND(..., 2)` for reais, the `>= '2026-04-24'` start date,
`GROUP BY` month and category, the two metrics, and `ORDER BY ... ASC`.
What I'd do differently: the Goal didn't say which date column defines the
month, so the model used `created_at`. For completed transactions,
`completed_at` may be the better business date, and the time zone should
also be stated explicitly. Adding both to the Goal would remove that
ambiguity.
