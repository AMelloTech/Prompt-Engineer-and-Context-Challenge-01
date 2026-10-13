---
question: 7
title: "Runbook-For-Recurring-Alert"
framework: "R-I-S-E (Role-Input-Steps-Expectation)"
provider: "Anthropic"
model: "Claude Opus 5.5"
effort: "default"
status: "final"
---

## Scenario

About 4 times a week, Beacon fires the same alert in the on-call channel:
`[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)`. Whoever
is on call spends 30–40 minutes resolving it, and the time varies a lot
because there's no documented procedure. Lorraine Baines wants a runbook any
on-call engineer can follow from start to finish without depending on someone
who knows the system (see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
ROLE: You are a senior SRE who writes runbooks for on-call engineers with
no prior knowledge of the system.

INPUT:
Recurring alert from Beacon (~4x/week, takes 30–40 min to resolve, no
documented procedure):
[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)

Environment:
- Chronos runs on EKS, namespace production, 6 replicas, HPA (min 4,
  max 12, CPU target 70%)
- Deployed by Argo CD from repo hvt/chronos-api
- Direct dependencies: Ledger (PostgreSQL) and Reactor (SQS queues)
- Metrics at /metrics, logs centralized in Beacon, dashboards in Grafana
- On-call tools: kubectl, aws cli, argocd cli
- On-call channel: #oncall-chronos
- Escalation: @chronos-core (response SLA 15 min business hours, 30 min
  outside)

STEPS:
1. Start with confirming the alert and checking pod memory.
2. Check whether a recent deploy caused it.
3. Check the dependencies (Ledger and Reactor).
4. Decide whether to escalate or mitigate.
5. Define when the incident can be closed.

EXPECTATION: A runbook in Markdown that any on-call engineer can follow
end to end. Each diagnostic step must have the exact commands to run and
the expected result. Include objective criteria to escalate to
@chronos-core and the criterion to close the incident.
```

## Model

**Model used:** Claude Opus 5.5 (Anthropic)

**Why this model:** the runbook has to combine kubectl, argocd, and aws
commands correctly and stay consistent across many steps, which favors a
strong reasoning model.

## Output

# Runbook: Chronos API — High memory usage

| | |
|---|---|
| **Alert** | `[CRITICAL] High memory usage on Chronos API pods (>85% for 10min)` |
| **Source** | Beacon → `#oncall-chronos` |
| **Service** | Chronos API (EKS, namespace `production`) |
| **Escalation** | `@chronos-core`: responds within 15 min (business hours) / 30 min (outside) |
| **Time budget** | Escalate if there's no diagnosis 30 min after the alert |

> **Assumptions to confirm once:** Deployment, label, and Argo CD
> application are all named `chronos-api` (label `app=chronos-api`), and the
> container listens on port 8080. Adjust the commands if they differ.

### Step 0: Acknowledge and set up (2 min)

Post in `#oncall-chronos`:

```
:eyes: Ack — investigating "High memory usage on Chronos API pods". Updates in this thread.
```

```bash
kubectl config current-context        # must point to the production EKS cluster
export NS=production
```

**Expected:** the context is the production cluster. If it isn't, switch to
it before continuing.

### Step 1: Confirm the alert and check pod memory (5 min)

```bash
kubectl -n $NS get pods -l app=chronos-api -o wide
kubectl -n $NS top pods -l app=chronos-api --sort-by=memory
kubectl -n $NS get deploy chronos-api \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

Compare each pod's memory against its memory **limit** (third command).

**Expected:** all pods `Running`/`Ready`, and you can see which pods are
above 85% of the limit. Write down the pattern:
- **One or two pods high, the rest normal** → probably a problem local to
  those pods (leak or a hot pod). Go to Step 4, mitigation A.
- **Most or all pods high** → load or a code regression. Continue to Step 2.

Check for restarts and OOM kills:

```bash
kubectl -n $NS get pods -l app=chronos-api \
  -o custom-columns=NAME:.metadata.name,RESTARTS:.status.containerStatuses[0].restartCount,LAST_TERMINATION:.status.containerStatuses[0].lastState.terminated.reason
```

**Expected:** `RESTARTS` 0 and no `OOMKilled`. Any `OOMKilled` means pods are
already hitting the limit. Note how many.

Check the autoscaler:

```bash
kubectl -n $NS get hpa chronos-api
```

**Expected:** the HPA target shows **CPU** at 70%. **Important:** this HPA
scales on CPU only. If CPU is below 70% while memory is above 85%, the HPA
**will not add pods**, so don't wait for it to fix the alert on its own.

### Step 2: Check whether a recent deploy caused it (5 min)

```bash
argocd app get chronos-api
argocd app history chronos-api
kubectl -n $NS rollout history deployment/chronos-api
```

**Expected:** the app is `Synced` and `Healthy`. Write down when the last
deploy happened.
- **Deploy in the last 24h and memory started rising after it** → likely
  regression. This is an escalation trigger (see Step 4). Don't roll back on
  your own.
- **No recent deploy** → continue to Step 3.

### Step 3: Check the dependencies (5 min)

**Ledger (PostgreSQL)**: look for connection or timeout errors in Chronos
logs:

```bash
kubectl -n $NS logs -l app=chronos-api --since=15m --tail=300 \
  | grep -iE "timeout|connection|pool|postgres|ledger"
```

**Expected:** no recurring errors. A burst of timeouts or pool errors means
Chronos may be holding requests in memory while waiting on Ledger. Confirm it
on the Ledger panel in Grafana.

**Reactor (SQS)**: check the queue backlog:

```bash
aws sqs list-queues --queue-name-prefix reactor
aws sqs get-queue-attributes --queue-url <REACTOR_QUEUE_URL> \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

Run the second command twice, 2 minutes apart.

**Expected:** the backlog is stable or falling. A growing backlog means
Reactor is slow, and Chronos may be buffering in memory.

**Process metrics** (optional, for the escalation report):

```bash
kubectl -n $NS port-forward pod/<POD_NAME> 8080:8080 &
curl -s localhost:8080/metrics | grep -iE "memory|heap|rss"
kill %1
```

### Step 4: Mitigate or escalate

#### Mitigation A: one or two pods high

Delete **one pod at a time** and wait for its replacement:

```bash
kubectl -n $NS delete pod <POD_NAME>
kubectl -n $NS get pods -l app=chronos-api -w   # wait for the new pod to be Ready
```

**Expected:** the new pod is `Ready` within 2 min and its memory is normal.
Only then move to the next high pod. **Never delete more than one pod at the
same time.**

#### Mitigation B: most pods high, no recent deploy, dependencies normal

Do a rolling restart (it respects the rollout strategy, so capacity is
kept):

```bash
kubectl -n $NS rollout restart deployment/chronos-api
kubectl -n $NS rollout status deployment/chronos-api --timeout=10m
```

**Expected:** `successfully rolled out` and memory below 70% on all pods
within 10 min.

> Don't edit the Deployment or HPA by hand (`kubectl edit`/`scale`). Argo CD
> will revert it, and the change won't be in `hvt/chronos-api`.

#### Escalation criteria: escalate to `@chronos-core` if **any** is true

| # | Criterion |
|---|---|
| E1 | Memory still above 85% on any pod **15 min after** a mitigation |
| E2 | **More than 2 `OOMKilled`** across pods in 30 min |
| E3 | **3 or more pods** above 85% at the same time and mitigation B didn't bring them below 70% |
| E4 | A **deploy in the last 24h** correlates with the start of the memory increase |
| E5 | **Ledger errors** or a **growing SQS backlog** for 10+ min |
| E6 | **30 min since the alert** with no clear cause |
| E7 | Users are affected (5xx errors or latency rising on the Grafana dashboard) |

Escalation message for `#oncall-chronos`:

```
@chronos-core escalating "High memory usage on Chronos API pods"
- Criterion: E<n>
- Pods affected: <n>/<total>, max memory: <x>% of limit
- OOMKilled in last 30 min: <n>
- Last deploy: <time / revision>
- Ledger/SQS: <normal / what you saw>
- Actions taken: <mitigation A/B, time>
```

If there's no response within the SLA (15 min business hours / 30 min
outside), mention `@chronos-core` again and notify Lorraine Baines (SRE
lead).

### Step 5: Close the incident

Close **only when all** of these are true:

- [ ] All pods below **85%** of the memory limit for **30 consecutive minutes**
- [ ] No new restarts or `OOMKilled` in that window
- [ ] Ready replicas ≥ HPA minimum (4)
- [ ] The alert is resolved in Beacon

Closing message in the thread:

```
:white_check_mark: Resolved — <cause, if known>. Action: <what was done>. Stable for 30 min.
```

Since this alert fires about 4 times a week, also open a follow-up ticket
for `@chronos-core` to investigate the root cause (possible memory leak) and
to evaluate adding a **memory** metric to the HPA.

## Rationale

**Role** ("senior SRE who writes runbooks for on-call engineers with no
prior knowledge") set the tone: every step has copy-paste commands, a time
budget, and plain-language "Expected" checks. **Input** carried every fact
from the brief, and that's what let the model notice that the HPA scales
on CPU while the alert is about memory (Step 1), even though no step asked
for it. **Steps** gave the runbook's skeleton: confirm → deploy →
dependencies → mitigate/escalate → close. **Expectation** required exact
commands, expected results, objective escalation criteria, and a closure
criterion, which became the E1–E7 table and the closing checklist. What
I'd do differently: the prompt asked for "objective criteria" without
defining them, so the thresholds (15 min, 2 OOMKills, 3 pods) were invented
by the model and still need to be agreed with Lorraine and `@chronos-core`.
The prompt also gave no safety rules, and the model added "one pod at a
time" and "don't edit by hand" on its own. Stating thresholds, the HPA
check, and guardrails in the prompt would make the result less dependent
on the model's judgment.
