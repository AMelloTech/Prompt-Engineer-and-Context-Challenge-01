---
question: 5
title: "Legacy-Deploy-Modernization"
framework: "B-A-B (Before-After-Bridge)"
provider: "OpenAI"
model: "GPT 6 Luna"
effort: "extra high"
status: "final"
---

## Scenario

During a production review, Doc Brown pulled the Chronos manifest and found a
Deployment George McFly wrote three years ago. Nobody has touched it since,
and much of what the company standard now requires is missing. Modernizing it
landed on our desk (see the general context in
[`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
Before: We have a manifest that was writing 3 years ago and since then now one has touched it. We need to modernize it.
After: The modernized version needs to have high availability, versioned images, secrets outside the manifest, resource request, limits, liveness, readiness probe, securityContext not root and the other best practices
Bridge: Re-write this manifest and deliver a modernized version of it. apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

## Model

**Model used:** GPT 6 Luna (OpenAI), extra high effort

**Why this model:** the task touches many Kubernetes fields at once (HA,
probes, security context, secrets), and the highest reasoning effort lowers
the chance of an incomplete or inconsistent manifest.

## Output

Below is a multi-document manifest with three replicas, a Service, disruption
protection, probes, resource settings, and a non-root security context. The
image tag and health-check paths are examples; update them to match your
published image and application endpoints.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 5
  progressDeadlineSeconds: 600
  selector:
    matchLabels:
      app: chronos-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chronos-api
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chronos-api
      containers:
        - name: api
          # Replace with a real published version; pin by digest where possible.
          image: chronos-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: JWT_SECRET
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  type: ClusterIP
  selector:
    app: chronos-api
  ports:
    - name: http
      port: 80
      targetPort: http
      protocol: TCP
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
```

Provision the chronos-api-secrets Secret separately through your
secret-management process; its keys must be DB_PASSWORD and JWT_SECRET. The
resource values are starting points to tune against observed usage. Since the
original credentials were included in the manifest you shared, rotate them
when migrating.

## Rationale

**After** carries every requirement from the brief, and each maps to the
output: HA (`replicas: 3`, PodDisruptionBudget, zone/host
`topologySpreadConstraints`, `maxUnavailable: 0`), a versioned image
(`chronos-api:1.0.0`), secrets via `secretKeyRef`, requests/limits,
liveness/readiness probes, and a non-root `securityContext`. The open-ended
"other best practices" produced extras like `readOnlyRootFilesystem`,
dropped capabilities, seccomp, and a startup probe. **Bridge** ("re-write...
deliver a modernized version") yielded a multi-document YAML, and the model
flagged credential rotation on its own. **Before** was the weak part: it only
said the manifest was old, and the legacy YAML sat under Bridge instead of
Before. What I'd do differently: move the manifest into Before and list its
concrete problems (single replica on the company's gateway, `:latest`,
plaintext credentials), and in After say that secrets come from AWS Secrets
Manager. Without that, the model fell back to a plain Kubernetes Secret. It
also had to invent the image tag and probe paths, which the prompt could
have supplied or marked as placeholders.
