---
title: Deployment
parent: Advanced
nav_order: 50
---

# Deployment

Ductwork is designed to run reliably in production environments, from traditional servers to containerized deployments. By default, bin/ductwork starts a supervisor that forks child processes: one pipeline advancer and one job worker per configured pipeline. This works well for traditional deployments but conflicts with container best practices.

**The container problem**: Containers expect a single foreground process. Forking introduces complications with signal handling, logging, and process management; container orchestrators like Kubernetes work best when they can directly manage each process.

## The `role` Configuration

Ductwork provides a `role` configuration that allows you to run a single process type without forking. This aligns with the container philosophy of one process per container.

Set the role in your YAML configuration:

```yaml
# config/ductwork.pipeline_advancer.yml
default: &default
  role: advancer
  pipelines: "*"

production:
  <<: *default
```

```yaml
# config/ductwork.job_worker.yml
default: &default
  role: worker
  pipelines:
    - EnrichUserDataPipeline
    - ProcessOrdersPipeline

production:
  <<: *default
```

Or override via environment variable:

```bash
DUCTWORK_ROLE=advancer bin/ductwork
```

## Docker and Containerized Deployments

### Container-per-Role Strategy

Run separate containers for each role. This gives you independent scaling, cleaner logs, and better resource isolation.

**Dockerfile:**

```dockerfile
FROM ruby:3.3-slim

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install --without development test

COPY . .

CMD ["bin/ductwork"]
```

**docker-compose.yml:**

```yaml
version: "3.8"

services:
  pipeline_advancer:
    build: .
    command: bin/ductwork -c config/ductwork.yml
    environment:
      DUCTWORK_ROLE: advancer
      RAILS_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
    depends_on:
      - db
    restart: unless-stopped

  job_worker_enrichment:
    build: .
    command: bin/ductwork -c config/ductwork.enrichment.yml
    environment:
      DUCTWORK_ROLE: worker
      RAILS_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
    depends_on:
      - db
      - pipeline_advancer
    deploy:
      replicas: 3
    restart: unless-stopped

  job_worker_orders:
    build: .
    command: bin/ductwork -c config/ductwork.orders.yml
    environment:
      DUCTWORK_ROLE: worker
      RAILS_ENV: production
      DATABASE_URL: postgres://db:5432/myapp
    depends_on:
      - db
      - pipeline_advancer
    deploy:
      replicas: 2
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Why Separate Containers?

**Scaling:** Scale job workers independently of the advancer. If your bottleneck is step execution, add more worker containers without spinning up redundant advancers.

**Resource allocation:** Assign different CPU and memory limits based on workload characteristics. Advancers are typically lightweight; workers may need more resources depending on your step logic.

**Logging:** Each container's stdout is a single, coherent log stream. No interleaved output from multiple processes.

**Failure isolation:** A crash in a job worker doesn't affect the advancer or other workers. The orchestrator restarts only what failed.

---

## Kubernetes Deployment

Kubernetes deployments follow the same container-per-role pattern.

### Pipeline Advancer Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ductwork-advancer
  labels:
    app: ductwork
    component: advancer
spec:
  replicas: 1  # Usually only need one
  selector:
    matchLabels:
      app: ductwork
      component: advancer
  template:
    metadata:
      labels:
        app: ductwork
        component: advancer
    spec:
      containers:
        - name: advancer
          image: myapp:latest
          command: ["bin/ductwork"]
          args: ["-c", "config/ductwork.yml"]
          env:
            - name: DUCTWORK_ROLE
              value: advancer
            - name: RAILS_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            exec:
              command:
                - pgrep
                - -f
                - ductwork
            initialDelaySeconds: 10
            periodSeconds: 30
          terminationGracePeriodSeconds: 45
```

### Job Worker Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ductwork-worker
  labels:
    app: ductwork
    component: worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: ductwork
      component: worker
  template:
    metadata:
      labels:
        app: ductwork
        component: worker
    spec:
      containers:
        - name: worker
          image: myapp:latest
          command: ["bin/ductwork"]
          args: ["-c", "config/ductwork.yml"]
          env:
            - name: DUCTWORK_ROLE
              value: worker
            - name: RAILS_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
          livenessProbe:
            exec:
              command:
                - pgrep
                - -f
                - ductwork
            initialDelaySeconds: 10
            periodSeconds: 30
          terminationGracePeriodSeconds: 45
```

### Horizontal Pod Autoscaling

Scale workers based on CPU utilization or custom metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ductwork-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ductwork-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Health Checks

For orchestrators that support health checks, you can verify the Ductwork process is running:

**Simple process check:**
```bash
pgrep -f ductwork
```

**Heartbeat-based check:** Query the `ductwork_processes` table for recent heartbeats from the current container. A process that hasn't reported a heartbeat in 5 minutes is considered unhealthy.

```ruby
# Example health check endpoint
Ductwork::Process
  .where('last_heartbeat_at > ?', 5.minutes.ago)
  .exists?
```

## Traditional (Non-Containerized) Deployment

For VMs or bare-metal servers, use the default supervisor mode with a process manager like systemd:

```ini
# /etc/systemd/system/ductwork.service
[Unit]
Description=Ductwork Pipeline Processor
After=network.target postgresql.service

[Service]
Type=simple
User=deploy
WorkingDirectory=/var/www/myapp/current
ExecStart=/var/www/myapp/current/bin/ductwork -c config/ductwork.yml
Restart=always
RestartSec=5
Environment=RAILS_ENV=production

[Install]
WantedBy=multi-user.target
```

In this mode, the supervisor handles process management internally, restarting crashed children automatically.

## Summary

| Deployment Type | Role Config | Process Management |
|-----------------|-------------|-------------------|
| Traditional (VM/bare-metal) | `supervisor` (default) | systemd, upstart, etc. |
| Docker/Kubernetes | `advancer` or `worker` | Orchestrator handles restarts |

The `role` configuration lets you choose the model that fits your infrastructure. For containers, run separate services for each role. For traditional deployments, let the supervisor manage child processes.
