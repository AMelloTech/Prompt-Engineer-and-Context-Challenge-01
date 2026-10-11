---
question: 3
title: "Report-Of-Cost-Reduction-For-Cloud"
framework: "T-A-G (Task-Action-Goal)"
provider: "Anthropic"
model: "Claude Opus 5"
effort: "high"
status: "final"
---

## Scenario

Goldie Wilson set the metric for next semester: a 15% cloud cost reduction by
the end of the period, without degrading the SLA. A FinOps analysis of the
current AWS cost report is needed to find where that reduction can come from
(see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
TASK: You are a FinOps specialist and we need to delivery a 15% cost reduction in cloud in the next period. The cloud provider we use is AWS.
ACTION: Using this report find oportunities to reduce cost that could drive us to the 15% reduction we need without degrading the SLA.

servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

GOAL:This report should be order by economic oportunities with less impact, how much of reduction it would delivery in percentagem of the total count, and the effort to implement it (low, medium, high) and the risks or pre requists needed.
```

## Model

**Model used:** Claude Opus 5 (Anthropic), high effort

**Why this model:** the task requires arithmetic over 12 cost lines plus
multi-criteria ranking (impact, effort, risk) in a single pass — a reasoning
model handles the cumulative-percentage math and the SLA constraint more
reliably than a lighter one.

## Output

### Baseline

Current total: **$41,800/month** ($501,600/year). The 15% target =
**$6,270/month** ($75,240/year) in reduction.

### Opportunities ordered by lowest operational impact

#### Tier 1 — Low effort, no SLA risk

| # | Action | Savings/mo | % of total | Effort | Risks / Prerequisites |
|---|---|---|---|---|---|
| 1 | **CloudWatch Logs**: retention 90d → 30d, exporting audit log groups to S3 Glacier IR | $1,450 | 3.47% | Low | Validate with Strickland whether compliance requires 90d; move the audit trail to S3 before reducing |
| 2 | **S3 Standard → Intelligent-Tiering** + lifecycle for cold data | $930 | 2.22% | Low | Run S3 Storage Lens first; objects <128KB don't benefit from auto-tiering |
| 3 | **EBS gp3**: reduce over-provisioned IOPS/throughput + delete orphaned snapshots | $250 | 0.60% | Low | Confirm IOPS baseline in CloudWatch over 14 days |
| 4 | **Lambda** → Graviton (arm64) + memory tuning | $200 | 0.48% | Low | Recompile native dependencies; validate cold start |
| 5 | **CloudWatch Metrics**: drop custom metrics not backing a dashboard/alarm | $180 | 0.43% | Low | Audit which metrics feed Beacon's alerts before removing |
| | **Tier 1 subtotal** | **$3,010** | **7.20%** | | |

#### Tier 2 — Medium effort, required to close the target

| # | Action | Savings/mo | % of total | Effort | Risks / Prerequisites |
|---|---|---|---|---|---|
| 6 | **EC2 on-demand** (45% util): right-sizing + 1-year Compute Savings Plan over the stable baseline | $2,050 | 4.90% | Medium | 1-year commitment — requires 30-day baseline analysis; keep on-demand for variable overflow |
| 7 | **RDS PostgreSQL**: migrate to Graviton class (r7g) + Reserved Instance | $1,400 | 3.35% | Medium | **Keep multi-AZ** (SLA requirement); validate engine version compatibility; run in a maintenance window |
| | **Tier 2 subtotal** | **$3,450** | **8.25%** | | |

> **Tier 1 + Tier 2 = $6,460/month = 15.45% — target met** ✅

#### Tier 3 — Buffer / later phases

| # | Action | Savings/mo | % of total | Effort | Risks / Prerequisites |
|---|---|---|---|---|---|
| 8 | **EKS** (58% util): bin-packing via Karpenter, Graviton nodes, Spot in non-prod only | $1,340 | 3.21% | High | Spot **not** for core workloads; requires PodDisruptionBudgets and drain testing |
| 9 | **VPC Gateway Endpoints** (S3/DynamoDB, free) to cut NAT processing + Data Transfer | $700 | 1.67% | Medium | **Do not consolidate the 3 NAT Gateways** — one per AZ preserves resilience and the SLA |
| 10 | **ElastiCache Redis** (40% util): right-size node family | $630 | 1.51% | Medium | Preserve failover headroom; test under real peak |
| | **Tier 3 subtotal** | **$2,670** | **6.39%** | | |

### Out of scope this cycle

**EC2 reserved** ($4,200, 72% util) — already under a 1-year contract. No
action until renewal; evaluate converting to a Savings Plan at that date.

### Recommendation

Execute **Tier 1 immediately** (7.20%, no SLA risk) and **Tier 2 in parallel**
with the listed validations — that closes 15.45%. Tier 3 stays as a 6.39%
buffer in case a Tier 2 assumption doesn't hold, giving comfortable margin
over the target.

## Rationale

**Task** establishes the role and the hard numeric constraint ("FinOps
specialist", "15% cost reduction", "AWS"), which made the model anchor every
estimate to a computed baseline ($41,800/mo) instead of generic advice.
**Action** supplies the raw material — the 12-line cost report with
utilization percentages — and the non-negotiable boundary ("without degrading
the SLA"), visible in the output where it refuses to consolidate the NAT
Gateways and insists on keeping RDS multi-AZ. **Goal** dictates the exact
output shape (ordered by least impact, % of total, effort low/medium/high,
risks/prerequisites), which produced the tiered table and the cumulative
total showing the target was reached at 15.45%.
