---
question: 1
title: "Docker-Para-O-Lift"
framework: "R-T-F (Role-Task-Format)"
provider: "Anthropic"
model: "Claude Sonnet 5"
effort: "medium"
status: "final"
---

## Scenario

Lift, Hill Valley Tech's beta product, is leaving the VMs and moving to run
on a Kubernetes cluster. Before the migration, the application's Dockerfile
needs to be created (see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
Role: As a senior DevOps Engineer, specialized in Docker and Kubernetes
Task: Create a dockerfile following all the best practices using the structure that is already created for the project

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py

and the requirement.txt file

Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9

we need 2 env variable present in the runtime DATABASE_URL and API_KEY. Remember that in production the service run with --bind 0.0.0.0:8000 --workers 4 app:app
Format: Dockerfile
```

## Model

**Model used:** Claude Sonnet 5 (Anthropic)

**Why this model:** good coverage of well-known Docker/Python best practices (multi-stage build, non-root user, `pip --user`) without needing external tools, well suited for a well-scoped configuration-artifact generation task.

## Output

```dockerfile
# syntax=docker/dockerfile:1

# ---- Build stage ----
FROM python:3.12-slim AS builder

WORKDIR /app

RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ---- Runtime stage ----
FROM python:3.12-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd -r lift && useradd -r -g lift -d /app lift

WORKDIR /app

COPY --from=builder /root/.local /home/lift/.local
COPY --chown=lift:lift app.py requirements.txt ./
COPY --chown=lift:lift lib/ ./lib/

ENV PATH=/home/lift/.local/bin:$PATH \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Required at runtime — injected via orchestrator secrets/config, not baked into the image
ENV DATABASE_URL="" \
    API_KEY=""

USER lift

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:app"]
```

## Rationale

**Role** ("senior DevOps Engineer, specialized in Docker and Kubernetes")
anchors the model in production best practices — multi-stage build,
non-root user, layer reduction, and a `slim` base image. **Task** specifies
the project's file structure, the exact `requirements.txt`, and the two
required env vars (`DATABASE_URL`, `API_KEY`), plus the production command
(`gunicorn --bind 0.0.0.0:8000 --workers 4 app:app`), all of which appear
literally in the final `CMD`. **Format** ("Dockerfile") constrained the
output to a single code artifact, with no surrounding explanation.

