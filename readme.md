# AKS & Databricks Migration — Case Study

## Problem Statement

The organisation currently operates **30+ application workloads** across on-premises servers and maintains **40+ pipeline jobs** on Databricks with manual code uploads and no automated testing. Across **25+ repositories**, the engineering organisation lacks:

| Gap | Impact |
|---|---|
| No cloud-native application platform | Cannot scale, lacks HA, and on-prem schedulers are fragile |
| Manual Databricks deployments | Error-prone releases, no testing gates, inconsistent environments |
| Mixed compute estate | Container instances, legacy VMs, on-prem servers — no unified operating model |

This case study proposes a structured migration to **Azure Kubernetes Service (AKS)** and an **automated Databricks CI/CD platform**, underpinned by GitOps principles and Infrastructure as Code.

---

## Migration Operating Model

![Migration Operating Model](images/majorstreamoperatingmodel.jpg)

The migration is organised into three sequential operating streams, each delivering distinct outcomes:

| Stream | Goal |
|---|---|
| **Platform Foundation** | Build Dev/Stg/Prod environments and a GitOps backbone on AKS |
| **Databricks Automation** | Eliminate manual uploads and establish a data pipeline release process for all 40+ jobs |
| **Workload Migration** | Move applications in waves from on-prem / mixed compute to AKS |

---

## Programme Roadmap

![Product Roadmap](images/productroadmap.jpg)

The full programme spans seven phases:

### Phase 1 — Discovery & Assessment
- Standardise naming conventions: workload, service, zone, environment, instance
- Establish baseline governance policies on Azure: OPA (Spacelift/Gatekeeper), RBAC, cluster standards, network policies
- Define Release & Deployment Strategy using Conventional Commits and semantic versioning
- Create Golden Container Images — Alpine Distroless base images shared across 25+ repositories
- Define reusable testing strategy: unit, component, integration, and data validation workflows

### Phase 2 & 3 — Build Platform Foundation
- Bootstrap Azure Landing Zones: CIDR planning, Management Groups, IAM for self-hosted runner infra
- Build Hub & Spoke VNet topology with WAF, DNS registry, and runner infrastructure
- Deploy Observability layer: Azure Monitor + Grafana
- Provision AKS clusters (Dev/Stg/Prod) via IaC (Pulumi/Terraform): NSGs, Key Vaults, App Config, Managed Identities
- Bootstrap GitOps (Argo CD) onto each cluster from the Platform Monorepo
- Create reusable deployment templates: application deploy, KSM secrets workflows, feature flags, ad-hoc runbooks
- Set up CI/CD template for Databricks

### Phase 4 & 5 — Begin Migration (On-Prem → Cloud)
- Migrate 2–3 low-risk apps to prove out AKS networking and GitOps flow
- Prefer already-containerised workloads first for lower friction
- Test deployment strategies: Rolling, Blue/Green, Canary, Load Balanced (L4/L7)
- Add infra deployment pipeline approval gates and automated testing gates
- Automate the most critical Databricks jobs

### Phase 6 — Mass Migration
- Migrate remaining 25+ on-prem applications in waves
- Consolidate all Container Instances into AKS
- Full automation of all 40+ Databricks pipelines
- Progressive decommissioning of on-prem servers after a stability soak window

### Phase 7 — Optimisation
- Implement HPA (Horizontal Pod Autoscaler) across services
- Self-Service Platform for application teams
- FinOps: cost controls, right-sizing, operation cost optimisation
- Dashboards & Alerting (SLOs, DORA metrics)

---

## Platform Architecture & Technology Decisions

### Target Platform Layer

![Workload Classification](images/workloadclassification.jpg)

The organisation moves to an **Azure-first platform** with clear layer assignments:

| Layer | Target |
|---|---|
| Compute | Azure Kubernetes Service (AKS) |
| Data Pipelines | Azure Databricks |
| Networking | Private Endpoints + VNet Integration |
| CI/CD | GitHub Actions |
| IaC | Pulumi or Terraform (org preference) |
| Secrets | Azure Key Vault |
| Config | Azure App Configuration |

### Workload Classification

| Type | Example | Target Platform |
|---|---|---|
| Stateless API | .NET Services | AKS |
| Batch / Scheduled Jobs | On-prem schedulers | AKS CronJobs |
| Data Pipelines | Databricks notebooks | Azure Databricks |
| Legacy Monoliths | Containerised and lifted | AKS |
| Low Complexity Scripts | Scripts | Azure Container Apps |

### Build vs Buy Decisions

![Build vs Buy](images/buildvsbuy.jpg)

| Capability | Choice | Build/Buy/Adopt | Rationale |
|---|---|---|---|
| IaC Control Plane | Spacelift | Buy | Policy-as-code, drift detection, multi-stack RBAC, strong governance for regulated environments |
| GitOps (AKS) | Argo CD | Adopt (open source) | Best-in-class declarative sync, drift detection, rollback |
| Networking/Security | Cilium + Hubble | Adopt (open source) | eBPF-based zero-trust, sidecarless, best-in-class east-west visibility and scaling |
| Observability | Azure Monitor + Grafana | Buy + Extend | Native Azure stack plus flexible visualisation |
| Telemetry | OpenTelemetry | Adopt | Open standard; unified tracing, logging, and metrics |
| Databricks Deployment | Databricks Asset Bundles (DAB) | Buy (Azure ecosystem) | Native CI/CD approach for Databricks |
| CI/CD | GitHub Actions / Azure DevOps | Buy either | GitHub Actions for OIDC-native flow; Azure DevOps for stricter approval/env/RBAC governance |
| IaC Tooling | Pulumi vs Terraform | Pulumi (adopt) / Terraform (buy) | Pulumi enables TDD and inner-source dev; Terraform is a more mature provider ecosystem |

---

## AKS Platform — Detailed Design

### Network Architecture

![AKS Architecture](images/aks/aksarchitecturenetwork.jpg)

The AKS platform runs across two Azure regions with a Hub & Spoke VNet topology:

- **Internet ingress** flows through Azure Front Door Premium + WAF (Edge TLS, health probes, failover)
- **Corporate access** comes through VPN / Express Route
- **Azure Private DNS Zone** (control plane) linked to each spoke VNet
- **Each region** has its own Spoke VNet containing:
  - AKS Cluster: Ingress/Gateway → Internal Load Balancer → Services (Cluster IP) → Pods
  - Private Endpoints subnet: Key Vault PE, App Config PE

### Separation of Concerns — IAC vs Service Deployment

![IAC vs Service Deployment Strategy](images/aks/iacvsservicedeploymentstrategy.jpg)

Two clear ownership boundaries are enforced:

**Platform Engineering (IAC Monorepo)**
- Owns: Pulumi/Terraform code for AKS clusters, VNet/Subnet, KV, App Config, Ingress, OIDC Identities, Argo CD, Cilium
- Pipeline: Validate → Plan → Approval → Apply to Azure

**Application Teams (Application Repo)**
- Owns: Source code, Dockerfile, Kustomize manifests, deployment overlays
- Pipeline: PR → Build → Test → Container Scan → Push to ACR → Update Kustomize tag → Commit to Deploy Repo → Argo CD syncs → AKS

Each service runs in its own namespace with Workload Identity scoped access to Key Vault and App Config.

### Pipeline Flows

![IAC vs Service Deployment Pipeline Flow](images/aks/iacvsservicedeploymentpipelineflow.jpg)

**IAC Pipeline:** `PR → Validate → Security Checks → Plan → Approve → Apply to Azure`

**Service Deployment Pipeline:** `PR → Build → Test → Container Scan → Push to ACR → Deploy Repo → Argo CD → AKS`

Argo CD provides pull-based GitOps deployment with drift detection and rollback. Cilium enforces east-west network policy with deny-by-default and full visibility.

**Repository Structure:**

```
# IAC Monorepo
/landing-zone   /networking   /aks   /identity
/acr   /kv   /app-config   /ingress   /observability
/argocd-bootstrap   /cilium

# Application Repo
/src   /test   /Dockerfile
/deploy/base
  deployment.yml   service.yml   ingress.yml   kustomization.yml
/deploy/overlays/dev   /deploy/overlays/stg   /deploy/overlays/prod
```

### Detailed Build Sequence — AKS

#### Phase 1 — Build Landing Zone

![Detailed Build Sequence Part 1](images/aks/detailedbuildsequencepart1.jpg)

**Environment Model:** Separate Azure subscription, VNet, Key Vault, App Config, and AKS cluster per environment (Dev/Stg/Prod).

**Platform Monorepo First Release provisions:**
resource groups, VNets & subnets, private DNS, ACR, Log Analytics / Azure Monitor, Key Vault, App Config, AKS clusters, User-Assigned Managed Identities (UAMI), GitOps bootstrap

**AKS Baseline:**
- OIDC issuer with Federated Identity (MS Entra Workload ID)
- Autoscaler with separate system and workload node pools
- Monitoring integration
- Ingress / Gateway API Foundation
- Azure CNI powered by Cilium

**Environment Differentiation:**

| | Dev | Staging & Prod |
|---|---|---|
| Node pools | Smaller | Same topology |
| HA | Lower | Full HA, zonal node pooling |
| SKU | Cheaper choices | Prod-grade |
| Cluster | Shared | Private cluster |
| RBAC | Relaxed policies | Strict RBAC |
| Ingress | Basic | Prod-grade edge path |

#### Phase 2 — Bootstrap GitOps

- Install Argo CD on each cluster from the Platform Monorepo
- Create base namespaces and shared platform services
- Define cluster add-ons as Git-managed resources: ingress/gateway, cert handling, external secrets, policy agents, observability agents

**Split of Responsibility:**

| Repo | Responsibility |
|---|---|
| Platform Repo | Creates clusters |
| GitOps (Argo CD) | Installs cluster-level shared services |
| Application Repos | Publish deployable artifacts |
| Cluster Config Repo | Decides what version runs where |

#### Phase 3 — Standardise Build Pipeline (25+ Repos)

For every application repository, add:
- PR validation, linting, unit tests
- Container build, image scanning, artifact versioning
- Publish to ACR, package Kustomize bundle

> GitOps consumes the built image tag from Git — not from a manual deployment action.

#### Phases 4–7 — Workload Migration Waves

![Detailed Build Sequence Part 2](images/aks/detailedbuildsequencepart2.jpg)

| Wave | Workload Type | Steps |
|---|---|---|
| **Wave 1** | Stateless low-risk APIs & workers | Containerise, externalise config, move secrets to KV via KSM, define k8s Deployment + Service, deploy dev → validate stg → promote prod |
| **Wave 2** | Scheduled jobs & on-prem schedulers | Isolate job logic, containerise, create CronJob spec, define concurrency & retry, add metrics/alerting, deploy via GitOps |
| **Wave 3** | Existing Container Instances & mixed compute | Compare runtime assumptions, move to standard base image, migrate deployment spec to Kustomize |
| **Wave 4** | Legacy / awkward workloads | Code refactoring where needed, transitional hybrid connectivity |

**Environment Promotion Model:**

| Environment | Behaviour |
|---|---|
| Dev | Rapid integration, frequent deploy, tests packaging & runtime model. Auto-sync on merge to main |
| Staging | Prod-like verification, smoke tests, dependency validation. Promotes same immutable artifact — no rebuild |
| Prod | Manual approval gate in Git. GitOps reconciles. Progressive strategy: rolling / canary / blue-green based on commit type |

---

## Generic Deployment Lifecycle

![Generic Deployment Lifecycle](images/lifecycle.jpg)

Applies to all workloads (AKS services and Databricks pipelines):

**On a branch (PR):**
- Run tests, linting, build image — do not deploy

**On PR approval:**
1. Lint/schema validation passed
2. Static vulnerability scanning passed
3. Unit tests passed
4. Component tests passed

**On merge to `main`:**
1. Version calculated via Conventional Commits
2. Immutable artifact built
3. Tagged (e.g., `v1.4.2`)
4. Artifact published to ACR / package registry / bundle store

**Deployment flow:** `DEV (single region, integration tests)` → `Staging Secondary` → `Staging Primary (integration tests)` → `manual approval gate` → `Prod Secondary` → `Prod Primary`

---

## Deployment Strategy

![Deployment Strategy](images/deploymentstrategy.jpg)

### Application Services (AKS)

| Scenario | Strategy |
|---|---|
| Non-breaking change | **Rolling Update** — readiness/liveness probes, gradual pod replacement |
| Breaking / high-risk change | **Blue/Green** — deploy to green (namespace or cluster), validate, switch traffic via AFD/App Gateway |
| Breaking / medium-low risk | **Canary** — traffic shift: 5% → 20% → 50% → 100% |

### Batch Jobs & Databricks Pipelines

| Scenario | Strategy |
|---|---|
| Non-breaking change | **Replace Version** — update CronJob/job definition, new run = new version |
| Breaking change | **Parallel Run (v1 + v2)** — same input, compare output, then switch scheduler to v2 |

### Universal — Applies to All Deployments

- **Observability & Governance:** OpenTelemetry / Azure Monitor for error and latency, Logs/Metrics/Alerts, DORA Metrics, Feature Flags via App Config

---

## Databricks — Detailed Design

### Architecture & Ownership

![Databricks Architecture](images/databricks/databricksarchitectureownership.jpg)

**Databricks Control Plane (Managed SaaS):**
Workspace UI, Jobs UI & Scheduler, Notebooks/Repos, REST API, Metadata & Orchestration — connected via Private Link / SCC / TLS.

**Azure Subscription (Organisation-owned) per environment (Dev/Stg/Prod):**
- VNet with Databricks subnets (VNet Injection): Driver VM, Worker VMs, Spark Job execution
- NAT Gateway / Firewall for controlled egress
- Data & Service Layer: ADLS/Blob, Cosmos/SQL, Key Vault, event streaming — all accessed via Private Endpoints, RBAC, and Managed Identities

**Ownership split:**

| Owner | Owns |
|---|---|
| Application Repos | Notebooks, Jobs, Pipeline code, Scheduling logic |
| IAC Monorepo | Workspace creation, VNet subnets, private endpoints, NAT/firewall, cluster policies, security baselines |

### Developer Workspace

![Databricks CI/CD Workspace](images/databricks/databrickscicdworkspace.jpg)

Developers (Python / Notebooks / SQL) commit code into a Git repository structured as:

```
/src             # pipeline code
/tests/          # unit and integration tests
databricks.yml   # Databricks Asset Bundle (DAB) definition
```

**Databricks Execution Model in Azure:**
`Workspace [Dev|Stg|Prd]` → `Databricks Job (defined by DAB)` → `Spark Cluster (Driver + Worker VMs)` → `ADLS / Delta Tables (Curated Data Outputs)`

### Databricks Automation — Detailed Project Plan

![Databricks Project Plan](images/databricks/databricksprojectplan.jpg)

#### Phase 1 — Foundation
- Create IAC repo: Databricks workspace (Dev first), cluster policies, RBAC access model
- Stand up Dev environment
- Define repository strategy: split repos per domain/workload
- Introduce basic CI pipeline: linting & unit tests for pipeline code
- Implement manual-triggered deployment to Databricks via CI
- Baseline monitoring & logging

#### Phase 2 — Promotion
- Expand IAC to Staging & Prod
- Introduce approval gates
- Artifact-based deployment using **Databricks Asset Bundles**
- Standardise job configuration templates and cluster usage patterns
- Integration testing in Staging
- Secrets management via Key Vault integration

#### Phase 3 — Scale
- Rollback and canary deployment strategy for pipelines
- Mass migration of all 40+ jobs
- Begin decommissioning of legacy manual processes

### IAC Pipeline for Databricks Workspaces

![Databricks IAC](images/databricks/databricksiac.jpg)

The Platform Engineering team owns a central IAC Monorepo that defines:
- Databricks workspace configuration
- Cluster policies, access/RBAC
- Networking/secrets, environment baselines

**Flow:** IAC Monorepo → IAC Pipeline (validate/plan/apply) → Dev workspace base → Staging (promoted base) → *(approval gateway)* → Prod (stable base)

All environments governed by Central Governance approvals and policy.

### Application CI/CD Pipeline for Databricks

![Databricks Application CI/CD](images/databricks/databricksapplicationcicd.jpg)

**Pipeline Flow:**

```
Git Repo (App Team CodeOwners)
  pipeline code + tests + databricks.yml
        │
        ▼
PR Pipeline: lint → unit test (pytest) → bundle validation
        │
        ▼
Merge to main: version & release bundle produced
        │
        ▼
Deploy bundle to Dev → run job → smoke test output
        │                    │
        │         Automated ephemeral Job Clusters
        │         (end-to-end validation, Dev/Stg workspace)
        ▼
Deploy to Staging → smoke test in Staging
        │
        ▼ approval gateway
Deploy to Prod → Monitor
```

---

## Appendix — Architecture Diagrams

| Diagram | Link |
|---|---|
| Migration Operating Model | [View](images/majorstreamoperatingmodel.jpg) |
| Programme Roadmap | [View](images/productroadmap.jpg) |
| Workload Classification & Platform Layers | [View](images/workloadclassification.jpg) |
| Generic Deployment Lifecycle | [View](images/lifecycle.jpg) |
| AKS Network Architecture | [View](images/aks/aksarchitecturenetwork.jpg) |
| IAC vs Service Deployment Strategy | [View](images/aks/iacvsservicedeploymentstrategy.jpg) |
| IAC vs Service Deployment Pipeline Flow | [View](images/aks/iacvsservicedeploymentpipelineflow.jpg) |
| Detailed Build Sequence — Part 1 (Phases 1–3) | [View](images/aks/detailedbuildsequencepart1.jpg) |
| Detailed Build Sequence — Part 2 (Phases 4–7) | [View](images/aks/detailedbuildsequencepart2.jpg) |
| Deployment Strategy | [View](images/deploymentstrategy.jpg) |
| Build vs Buy | [View](images/buildvsbuy.jpg) |
| Databricks Architecture & Ownership | [View](images/databricks/databricksarchitectureownership.jpg) |
| Databricks CI/CD Workspace | [View](images/databricks/databrickscicdworkspace.jpg) |
| Databricks Project Plan | [View](images/databricks/databricksprojectplan.jpg) |
| Databricks IAC Pipeline | [View](images/databricks/databricksiac.jpg) |
| Databricks Application CI/CD Pipeline | [View](images/databricks/databricksapplicationcicd.jpg) |
