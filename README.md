# CloudKitchen

[![CI](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](.github/workflows)
[![Go](https://img.shields.io/badge/Go-1.22-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)](https://react.dev)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS-326CE5?logo=kubernetes&logoColor=white)](https://aws.amazon.com/eks/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform&logoColor=white)](https://www.terraform.io)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

> A cloud-native, event-driven **food-delivery platform** built as 8 Go
> microservices plus a React frontend, deployed to **AWS EKS** via **GitOps**
> with a full observability and security baseline.

CloudKitchen is a portfolio-grade reference platform demonstrating microservice
design, async messaging, infrastructure-as-code, GitOps delivery, and
production-style monitoring/logging/security.

## What it is

- **8 backend microservices** (`auth`, `user`, `restaurant`, `menu`, `order`,
  `payment`, `delivery`, `notification`) written in Go, each listening on
  `:8080` and exposing `/metrics`, `/healthz`, `/readyz` with structured JSON logs.
- **React frontend** SPA served by nginx.
- **Sync** comms over REST, **async** comms over NATS JetStream events.
- Backed by **PostgreSQL**, **Redis**, and **NATS (JetStream)**.
- Shipped to **EKS** (`us-east-1`) through **GitHub Actions -> ECR -> ArgoCD**.

## Architecture

End-to-end view: source code in GitHub all the way through to a user's
browser hitting the running app, including the GitOps sync, TLS chain,
and observability fan-in.

```mermaid
flowchart TB
    %% ───────────── External actors ─────────────
    dev([Developer])
    user([End-user Browser])
    dns[GoDaddy DNS<br/>vijaygiduthuri.in]
    le[("Let's Encrypt ACME")]

    %% ───────────── Source / CI ─────────────
    gh[(GitHub Repo<br/>main branch)]
    actions[GitHub Actions<br/>build · Trivy scan · push]
    reg[(Container Registry<br/>ECR / GAR)]

    dev -->|git push| gh
    gh -->|workflow trigger| actions
    actions -->|push images| reg
    actions -->|bot commit:<br/>bump image tags| gh

    %% ───────────── Kubernetes cluster ─────────────
    subgraph cluster["Kubernetes cluster - EKS or GKE"]
        direction TB

        argo[ArgoCD<br/>App-of-Apps]

        subgraph traefik_ns["traefik namespace"]
            traefik[Traefik Ingress<br/>:80 and :443]
        end

        subgraph cm_ns["cert-manager namespace"]
            cm[cert-manager]
        end

        subgraph ck_ns["cloudkitchen namespace"]
            fe[React Frontend<br/>nginx]
            services["8 Go microservices on :8080<br/>auth, user, restaurant, menu,<br/>order, payment, delivery, notification"]
            pg[(PostgreSQL)]
            redis[(Redis)]
            nats{{"NATS (JetStream)"}}
        end

        subgraph mon_ns["monitoring namespace"]
            prom[Prometheus]
            grafana[Grafana]
            alert[Alertmanager]
        end

        subgraph log_ns["logging namespace"]
            promtail[Promtail DaemonSet]
            loki[Loki]
        end

        %% Runtime path-based routing
        traefik -->|/| fe
        traefik -->|/api/*| services
        traefik -->|/argocd| argo
        traefik -->|/grafana| grafana
        traefik -->|/prometheus| prom
        traefik -->|/alertmanager| alert

        %% Data plane
        fe -->|REST :8080| services
        services -->|SQL| pg
        services -->|cache · sessions| redis
        services <-->|pub / sub events| nats

        %% Cert provisioning
        cm -->|writes Secret<br/>cloudkitchen-tls| traefik

        %% GitOps sync (ArgoCD reconciles every platform App)
        argo -.->|syncs| traefik_ns
        argo -.->|syncs| cm_ns
        argo -.->|syncs| ck_ns
        argo -.->|syncs| mon_ns
        argo -.->|syncs| log_ns

        %% Observability fan-in
        prom -.->|scrape /metrics| services
        prom -.->|scrape| traefik
        promtail -.->|tail pod logs| loki
        grafana -.->|PromQL| prom
        grafana -.->|LogQL| loki
        prom -.->|fires alerts| alert
    end

    %% ───────────── External wiring ─────────────
    user -->|HTTPS| dns
    dns -->|A record| traefik
    cm <-->|HTTP-01 challenge<br/>renews every 75d| le

    gh -.->|ArgoCD polls / webhook| argo
    reg -.->|kubelet image pull| ck_ns

    classDef ext   fill:#fef3c7,stroke:#92400e,color:#1f2937
    classDef ci    fill:#dbeafe,stroke:#1e40af,color:#1f2937
    classDef data  fill:#fce7f3,stroke:#9d174d,color:#1f2937
    classDef obs   fill:#dcfce7,stroke:#14532d,color:#1f2937

    class dev,user,dns,le ext
    class actions,reg,argo ci
    class pg,redis,nats data
    class prom,grafana,alert,promtail,loki obs
```

**How to read this diagram**

- **CI/CD (top).** Developer pushes → GitHub Actions builds + Trivy-scans + pushes images to the container registry → a CI bot commits the new image tags back to `helm/cloudkitchen/values.yaml` → ArgoCD detects the diff (poll + webhook) and reconciles every platform App.
- **Cluster body (middle).** Traefik is the single ingress on `:80`/`:443`. cert-manager provisions the `cloudkitchen-tls` Secret via Let's Encrypt HTTP-01 (renews every 75 days). Traefik routes by path prefix: `/` → React frontend, `/api/*` → the 8 Go services, and `/argocd`, `/grafana`, `/prometheus`, `/alertmanager` → the respective platform UIs.
- **Data plane (cloudkitchen ns).** Services do sync REST on `:8080` to each other and PostgreSQL/Redis; async events flow over **NATS JetStream**.
- **Observability (bottom).** Prometheus scrapes `/metrics` from every pod; Promtail tails container logs into Loki; Grafana queries both. Alertmanager handles fired alerts.

See [`docs/architecture/PHASE-1.md`](docs/architecture/PHASE-1.md) for the full
design, event catalog, CI/CD, and GitOps flow.

## Tech stack

| Layer            | Technology |
|------------------|------------|
| Backend services | Go 1.22 (HTTP REST, Prometheus client, structured JSON logging) |
| Frontend         | React 18 + Vite, served by nginx |
| Data store       | PostgreSQL 16 |
| Cache / sessions | Redis 7 |
| Messaging        | NATS 2.10 + JetStream (event bus) |
| Containers       | Docker (per-service Dockerfiles) |
| Orchestration    | Kubernetes (AWS EKS) |
| Ingress / TLS    | Traefik + cert-manager (Let's Encrypt) |
| GitOps           | ArgoCD |
| Packaging        | Helm |
| IaC              | Terraform (VPC, EKS, ECR, IAM/IRSA) — `us-east-1` |
| CI/CD            | GitHub Actions (matrix build, Trivy gate, ECR push, values bump) |
| Metrics          | Prometheus (kube-prometheus-stack) + Grafana |
| Logging          | Loki + Promtail |
| Security scan    | Trivy (CI gate + optional trivy-operator) |

## Repository layout (flat monorepo)

```
cloudkitchen-app/
├── auth/            # Go service — authentication & JWT
├── user/            # Go service — user profiles
├── restaurant/      # Go service — restaurant management
├── menu/            # Go service — menu items
├── order/           # Go service — order lifecycle
├── payment/         # Go service — payments
├── delivery/        # Go service — delivery tracking
├── notification/    # Go service — notifications
├── frontend/        # React SPA
├── helm/            # Helm chart(s)
├── terraform/       # AWS infra (VPC, EKS, ECR, IAM/IRSA)
├── argocd/          # ArgoCD Applications (App-of-Apps)
├── monitoring/      # Prometheus + Grafana values & dashboards
├── logging/         # Loki + Promtail values
├── security/        # cert-manager, network policies, PSS, trivy, secrets
├── docker/          # docker-compose local stack
├── scripts/         # build / seed / port-forward / kubeconfig helpers
├── docs/            # architecture & docs index
├── .github/         # GitHub Actions workflows
└── README.md
```

## Quickstart (local, docker-compose)

```sh
# from the repo root
docker compose -f docker/docker-compose.yml up --build
```

Then:

| Component    | URL                     |
|--------------|-------------------------|
| Frontend     | http://localhost:3000   |
| auth         | http://localhost:8081   |
| user         | http://localhost:8082   |
| restaurant   | http://localhost:8083   |
| menu         | http://localhost:8084   |
| order        | http://localhost:8085   |
| payment      | http://localhost:8086   |
| delivery     | http://localhost:8087   |
| notification | http://localhost:8088   |
| NATS monitor | http://localhost:8222   |

Seed demo data (users per role, a restaurant, menu items, an order):

```sh
./scripts/seed.sh
```

Full local instructions: [`docker/README.md`](docker/README.md).

## Deployment overview

```mermaid
flowchart LR
    tf[Terraform] --> eks[(EKS us-east-1)]
    push[git push] --> gha[GitHub Actions]
    gha --> trivy[Trivy] --> ecr[(ECR)]
    ecr --> bump[bump helm/cloudkitchen/values.yaml] --> commit[commit]
    commit --> argo[ArgoCD auto-sync] --> eks
```

1. **Provision** infrastructure with **Terraform** (VPC, EKS, ECR, IAM/IRSA) in
   `us-east-1`.
2. **Bootstrap** the cluster: namespaces, Traefik, cert-manager, ArgoCD,
   kube-prometheus-stack, Loki/Promtail.
3. **CI** (GitHub Actions): matrix build per service -> **Trivy** scan ->
   push to **ECR** -> bump image tags in `helm/cloudkitchen/values.yaml` ->
   commit. **No `helm upgrade` in CI.**
4. **GitOps**: **ArgoCD** detects the committed change and **auto-syncs** the
   Helm release to EKS.

See the area guides:
[monitoring](monitoring/README.md) ·
[logging](logging/README.md) ·
[security](security/README.md) ·
[docs index](docs/README.md).

## License

MIT — see `LICENSE` (placeholder).
