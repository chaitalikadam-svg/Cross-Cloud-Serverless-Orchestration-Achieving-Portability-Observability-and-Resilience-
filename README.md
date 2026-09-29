# Cross-Cloud Serverless Orchestration

**Achieving Portability, Observability and Resilience Across AWS, Azure and Google Cloud**

A framework that deploys one OCI-compliant container image to **AWS Fargate, Azure Container Apps and Google Cloud Run**, then uses an observability-driven routing engine to send each request to the best provider in real time. It uses weighted latency, cost and reliability scoring, with a circuit breaker for automatic failover. It was built and evaluated on **real cloud infrastructure**, not simulation.

> **MSc Research Project (Cloud Computing), School of Computing, National College of Ireland, August 2026**
> **Author:** Chaitali Prakash Kadam (x24215449)
> **Supervisor:** Prof. Shreyas Setlur Arun

## Repositories

This project is split across two repositories:

| Repository | Purpose |
|---|---|
| [**Thesis_Project**](https://github.com/Chaitali-Kadam1008/Thesis_Project.git) | Infrastructure and delivery: Terraform modules for all three clouds, the benchmark container application, and the GitHub Actions CI/CD pipeline |
| [**orchestrator_layer**](https://github.com/Chaitali-Kadam1008/orchestrator_layer.git) | Orchestration and observability: the FastAPI routing engine, Prometheus, Grafana and the Locust load-testing harness, all run with Docker Compose |

**Order of setup:** build the infrastructure with `Thesis_Project` first, then point `orchestrator_layer` at the resulting application URLs.

---

## Table of Contents

- [Why this project](#why-this-project)
- [Research questions](#research-questions)
- [Architecture](#architecture)
- [How routing works](#how-routing-works)
- [Tech stack](#tech-stack)
- [Experimental setup](#experimental-setup)
- [Results](#results)
- [Getting started](#getting-started)
- [API reference](#api-reference)
- [CI/CD pipeline](#cicd-pipeline)
- [Limitations](#limitations)

---

## Why this project

Organisations that run on a single cloud face **vendor lock-in** and a **single point of failure**. Infrastructure tools such as Terraform can provision resources on several clouds, but they don't decide *at runtime* which provider should serve a request. Container-native serverless services (Fargate, Container Apps, Cloud Run) all accept standard OCI images, so in principle one image can run anywhere. This project adds the missing layer: a metrics-driven router that turns that portability into measurable cost, latency and reliability benefits.

## Research questions

> Can a cross-cloud serverless orchestration framework be designed to optimise cost, latency and reliability while ensuring portability and interoperability across AWS, Azure and Google Cloud?

1. **Portability:** can OCI-standard containerised functions be deployed unchanged across providers?
2. **Resilience:** does automated multi-cloud failover improve resilience compared with a single-cloud deployment?
3. **Observability:** what role does a multi-cloud observability layer play in routing accuracy, transparency and decision-making?

## Architecture
![Architecture Diagram](architecture-daigram(2).png)



The system has four loosely coupled layers:

1. **CI/CD pipeline:** builds one image and distributes it to every provider's registry, then provisions infrastructure with Terraform.
2. **Cross-cloud deployment layer:** three independent, identical copies of the workload. They know nothing about each other or about the router.
3. **Routing engine:** picks a provider per request and handles failover.
4. **Observability layer:** Prometheus and Grafana, which are both the evaluation tool and the live input to routing decisions.

The orchestration layer is deliberately hosted on a **separate EC2 instance**, outside all three benchmarked deployments, so the router's own host provider can't bias the results.

## How routing works

Each provider receives a score for every routing decision:

```
Score = 0.5 × latency score + 0.2 × cost score + 0.3 × reliability score
```

| Component | Source | Notes |
|---|---|---|
| **Latency** | Live 95th-percentile request latency queried from Prometheus | Observability data feeds the routing decision directly |
| **Cost** | Static cost per 1,000 invocations | AWS $0.0021, Azure $0.0049, GCP $0.0048 (0.5 vCPU / 1 GB, from published pricing) |
| **Reliability** | Exponentially weighted moving average of real forwarding successes and failures | Measured by the router, not self-reported by the provider |

Each component is min-max scaled across the three providers, so the best provider on a dimension scores 1.0 and the worst 0.0.

**Sticky routing:** the winning provider is kept for **60 seconds** before re-scoring, unless its circuit breaker opens. This avoids flip-flopping between close competitors.

**Circuit breaker (per provider):**

| State | Behaviour |
|---|---|
| `CLOSED` | Normal routing |
| `OPEN` | Opens when the forwarding error rate exceeds **5%** over a rolling 60 seconds |
| `HALF_OPEN` | After a **30-second** cooldown, one probe request is sent. **Three consecutive successes** close the circuit |

If a call to the chosen provider fails, the engine immediately retries on the next-best provider before returning an error to the client. Reliability is derived from the router's own request outcomes because a fully failed provider stops reporting metrics, so a self-reported health metric would go blind at exactly the wrong moment.

The weights are configurable through environment variables, so the router can be tuned without retraining or code changes.

## Tech stack

| Area | Technology |
|---|---|
| Compute | AWS Fargate (ECS + ALB), Azure Container Apps, Google Cloud Run |
| Container | Docker, OCI image (`python:3.11-slim`), Pillow, NumPy |
| Application and router | Python, FastAPI |
| IaC | Terraform (one root module, one module per provider) |
| CI/CD | GitHub Actions (AWS, Azure and GCP auth in one workflow) |
| Registries | Docker Hub, AWS ECR, Azure Container Registry, Google Artifact Registry |
| Observability | Prometheus, Grafana (dashboards as version-controlled JSON) |
| Load testing | Locust |
| Orchestration host | Amazon EC2 with Docker Compose |

## Experimental setup

- **Identical resources:** 0.5 vCPU and 1 GB memory per service, with a single warm instance each to isolate steady-state performance from cold-start variance.
- **Regions:** AWS `us-east-1`, Azure `eastus`, GCP `us-east4`, to reduce geographic latency as a confounder.
- **Benchmark workload** (decomposed into CPU, memory and I/O phases, following the Crossfit methodology):
  - **CPU-bound:** random image array generation, resize and grayscale conversion
  - **Memory-bound:** JPEG encoding into an in-memory buffer
  - **I/O-bound:** writing and reading the resulting images to disk
- **Load:** Locust with a random 1–2.5 s wait between requests, kept low to avoid queueing on the single instance behind each provider.
- **Provider labelling:** the only difference between deployments is the `CLOUD_PROVIDER` variable (`aws-fargate`, `azure-container-apps`, `gcp-cloud-run`). It labels all metrics and API responses.
- **Prometheus scrape targets (5):** three cloud deployments, the routing engine, and a client-side exporter in Locust. Comparing client-side and server-side latency shows network and ingress overhead per provider.

| Experiment | Purpose |
|---|---|
| **1. Uniform distribution baseline** | Equal traffic to each provider, bypassing the router, for a fair provider-neutral comparison |
| **2. Routed and failure injection** | All traffic through the router, then each provider disabled in turn (baseline, outage, recovery) |

## Results

### Experiment 1: cross-provider performance (about 1.3–1.6 req/s each)

| Metric | AWS Fargate | Azure Container Apps | GCP Cloud Run |
|---|---|---|---|
| Total API duration p95 | 400–800 ms, volatile | ~100 ms, stable | 200–400 ms |
| Total API duration p50 | 150–230 ms | 90–150 ms | 150–200 ms |
| Disk write p95 | 350–600 ms (highest) | 200–350 ms (lowest) | 250–450 ms |
| CPU transform p95 | 20–70 ms | 10–20 ms | 40–80 ms |
| Memory encode p95 | ~4.85 ms | 4.78–4.80 ms | up to 4.95 ms |
| Framework overhead p95 | ~4.76–4.82 ms | ~4.76 ms | ~4.76–4.82 ms |

Key takeaways:

- AWS's **median** latency is close to its peers, but its **tail latency** is far less predictable. That matters for reliability-sensitive routing.
- Stage-level breakdown points to **disk write time**, not CPU or memory, as the main driver of AWS's tail latency.
- Framework overhead is tiny (<0.1 ms spread between providers), and payload size is identical, which rules it out as a confounder.

### Experiment 2: controlled provider outages (2026-08-02, 12:30–12:59 UTC)

| Window | Provider | How it was disabled | Outage duration | Client success rate |
|---|---|---|---|---|
| 1 | GCP Cloud Run | Removed public invoker IAM binding | 4m 50s | **100%** |
| 2 | Azure Container Apps | Deactivated the active revision | 4m 54s | **100%** |
| 3 | AWS Fargate | Set ECS desired count to 0 | 7m 58s | **100%** |

- The failed provider dropped to **0 req/s** for the full outage.
- The two healthy providers absorbed the load, and traffic returned to a three-way split after recovery.
- The circuit breaker cycled between `OPEN` and `HALF_OPEN` at about 30-second intervals during each outage. Backend error rates were high, but **none reached the client**.
- AWS Fargate's longer window is consistent with slower container start-up compared with Azure's near-instant revision reactivation.

## Getting started

### Prerequisites

- Accounts on **AWS, Azure and GCP**, and a Docker Hub account
- Terraform, Docker and Docker Compose
- AWS CLI, Azure CLI and `gcloud`
- A GitHub repository with Actions enabled

### 1. Build the infrastructure (`Thesis_Project`)

```bash
git clone https://github.com/Chaitali-Kadam1008/Thesis_Project.git
cd Thesis_Project
```

1. Add the required secrets to the repository (see [CI/CD pipeline](#cicd-pipeline)).
2. Push to `main`. The workflow builds the image, pushes it to all registries and applies Terraform for AWS, then Azure, then GCP.
3. Note the three public application URLs from the Terraform outputs.

### 2. Deploy the orchestration layer (`orchestrator_layer`)

```bash
git clone https://github.com/Chaitali-Kadam1008/orchestrator_layer.git
cd orchestrator_layer
```

1. **Update the provider URLs** in the configuration to the three application URLs from step 1.
2. On the EC2 instance, add **inbound security-group rules** for the ports below.
3. Start the stack:

```bash
docker compose up -d
```

| Service | Port | Purpose |
|---|---|---|
| Routing engine | 8080 | `/invoke`, `/route`, `/metrics` |
| Locust web UI | 8089 | Load generation |
| Locust metrics exporter | 8001 | Client-side latency metrics (confirm against your compose file) |
| Prometheus | 9090 | Metrics and scrape targets |
| Grafana | 3001 | Dashboards |

### 3. Run the experiments

1. Open Locust at `http://<EC2_PUBLIC_IP>:8089`.
2. **Experiment 1:** start the three provider-specific user classes for a uniform baseline.
3. **Experiment 2:** start the routed user class, then disable providers one at a time:

```bash
# GCP Cloud Run: remove public access
gcloud run services remove-iam-policy-binding <service> --member=allUsers --role=roles/run.invoker

# Azure Container Apps: deactivate the active revision
az containerapp revision deactivate --name <app> --resource-group <rg> --revision <revision>

# AWS Fargate: scale the ECS service to zero
aws ecs update-service --cluster <cluster> --service <service> --desired-count 0
```

4. Restore each provider, then watch **Circuit Breaker State** and **Routed Requests per Second** in Grafana at `http://<EC2_PUBLIC_IP>:3001`.

## API reference

**Provider deployments (identical on all three clouds)**

| Endpoint | Description |
|---|---|
| `POST /invoke` | Runs the benchmark workload and returns JSON: `cpu_generation_ms`, `cpu_transform_ms`, `memory_encode_ms`, `disk_write_ms`, `disk_read_ms`, `total_internal_ms`, `output_bytes` |
| `GET /metrics` | Prometheus-format metrics labelled by `CLOUD_PROVIDER` |

**Routing engine (port 8080)**

| Endpoint | Description |
|---|---|
| `POST /invoke` | Scores providers, forwards to the winner, and fails over on error |
| `GET /route` | Shows the current score breakdown, circuit state and chosen provider, without forwarding a request |
| `GET /metrics` | Routing decisions, circuit-breaker transitions and per-provider scores |

## CI/CD pipeline

The GitHub Actions workflow runs on every push to `main`.

1. **Build:** one Docker image, pushed to Docker Hub as the canonical source. Later stages never rebuild it.
2. **Distribute:** each provider stage pulls the same image and re-tags it into ECR, ACR or Artifact Registry, which guarantees identical artifacts.
3. **Provision (sequential):** `deploy-fargate`, then `deploy-azure`, then `deploy-gcp`, each running `terraform apply -target=module.<provider>`. They run in sequence because they share one Terraform state file, and concurrent applies risk corrupting it.
4. **Idempotent import:** before each apply, the stage checks the live cloud (for example `aws ec2 describe-security-groups`, `az containerapp show`, `gcloud run services describe`) and runs `terraform import` for any existing resource missing from state. This prevents "already exists" failures.
5. **Force refresh:** because the image tag is pinned to `latest`, each stage explicitly forces a new deployment (`aws ecs update-service --force-new-deployment`, `az containerapp revision restart`, `gcloud run services update --image=...`).
6. **State persistence:** Terraform state is passed between jobs with `actions/cache`, keyed by `github.run_id`.

Authentication uses `aws-actions/configure-aws-credentials`, `azure/login` and `google-github-actions/auth` (GCP Workload Identity Federation) chained together. Configure repository secrets for Docker Hub, AWS, Azure and GCP credentials plus the deployment settings such as regions. Never commit credentials.

## Limitations

- Cost per 1,000 invocations uses **static estimates** from published pricing, because providers expose no reliable real-time per-invocation cost. They indicate relative ranking only.
- A **keepalive** mechanism keeps latency samples fresh for all providers. This trades some passive-observability purity for measurement stability.
- The orchestration layer runs on a **single EC2 instance**, so it is itself a single point of failure.
- Single warm instances and low load isolate steady-state performance but exclude cold-start and autoscaling behaviour.

Tool documentation: [AWS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/AWS_Fargate.html), [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/), [Google Cloud Run](https://cloud.google.com/run/docs), [Terraform](https://developer.hashicorp.com/terraform/docs), [Prometheus](https://prometheus.io/docs/), [Grafana](https://grafana.com/docs/), [Locust](https://docs.locust.io/), [Docker](https://docs.docker.com/), [GitHub Actions](https://docs.github.com/en/actions)
