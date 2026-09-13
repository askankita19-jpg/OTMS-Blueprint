# OTMS — Complete Project Blueprint

## 1. The project vision

We are building **one application — OTMS — and three alternative deployment architectures**, plus a fourth repository that provides monitoring and optional orchestration.

```text
                         ┌──────────────────────┐
                         │     OTMS APP         │
                         │                      │
                         │  Frontend            │
                         │  Employee API        │
                         │  Attendance API      │
                         │  Salary API          │
                         │  Notification API    │
                         └──────────┬───────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
       ┌───────────┐         ┌─────────────┐        ┌───────────┐
       │   OTMS    │         │ OTMS-Docker │        │ OTMS-EKS  │
       │           │         │             │        │           │
       │ EC2/AMI   │         │ Docker      │        │ Kubernetes │
       │ Terraform │         │ ECR         │        │ ECR       │
       │ Ansible   │         │ Jenkins     │        │ Terraform │
       │ Packer    │         │             │        │ EKS       │
       └─────┬─────┘         └──────┬──────┘        └─────┬─────┘
             │                      │                     │
             └──────────────────────┼─────────────────────┘
                                    │
                                    ▼
                          ┌───────────────────┐
                          │ OTMS-Monitoring   │
                          │                   │
                          │ Prometheus        │
                          │ Grafana            │
                          │ Alertmanager       │
                          │ Exporters          │
                          └───────────────────┘
```

### Core principle

> **Every deployment model must be independently deployable, testable, rollback-capable, and destroyable.**

And:

> **Monitoring must be independently deployable and must be able to attach to any one of the three deployment models.**

---

# 2. The four repositories

We will build these four repositories:

```text
github.com/askankita19-jpg/

├── OTMS
├── OTMS-Docker
├── OTMS-EKS
└── OTMS-Monitoring
```

Their responsibilities are deliberately different.

| Repository          | Purpose                                        |
| ------------------- | ---------------------------------------------- |
| **OTMS**            | Traditional VM/EC2 deployment                  |
| **OTMS-Docker**     | Docker-based deployment                        |
| **OTMS-EKS**        | Kubernetes/EKS deployment                      |
| **OTMS-Monitoring** | Monitoring + optional deployment orchestration |

---

# 3. The application itself

The application consists of five services.

```text
┌─────────────────────────────────────────────┐
│                  OTMS                       │
├─────────────────────────────────────────────┤
│                                             │
│ Frontend              React       :3000     │
│ Employee API          Go          :8080     │
│ Attendance API        Python      :8081     │
│ Salary API            Java        :8082     │
│ Notification API      Python      :8085     │
│                                             │
└─────────────────────────────────────────────┘
```

The original OTMS KT documents these five components and ports. 

Databases:

```text
PostgreSQL   :5432
Redis        :6379
ScyllaDB     :9042
```

Original application-to-database relationships:

```text
Employee API
 ├── ScyllaDB
 └── Redis

Attendance API
 ├── PostgreSQL
 └── Redis

Salary API
 ├── ScyllaDB
 └── Redis

Notification API
 └── ScyllaDB
```



---

# 4. Repository #1 — OTMS

## Purpose

This is our **traditional infrastructure deployment**.

We reproduce the original OTMS deployment model first, while fixing known issues where appropriate rather than deliberately carrying defects forward.

```text
Developer
    ↓
GitHub
    ↓
Jenkins
    ↓
Application CI
    ↓
Packer
    ↓
AMI
    ↓
Terraform
    ↓
Ansible
    ↓
EC2
    ↓
OTMS
```

The original architecture uses Jenkins, Shared Library, Job DSL, Terraform, Ansible and Packer in this overall workflow. 

---

## OTMS repository structure

Conceptually:

```text
OTMS/
│
├── applications/
│   ├── frontend/
│   ├── employee-api/
│   ├── attendance-api/
│   ├── salary-api/
│   └── notification-api/
│
├── Jenkins/
│   ├── CI_Pipelines/
│   ├── Packer_Pipelines/
│   ├── Ansible_CI_Pipelines/
│   ├── Ansible_CD_Pipelines/
│   ├── Terraform_CI_Pipelines/
│   ├── Terraform_CD_Pipelines/
│   ├── Terraform_Destroy_Pipelines/
│   └── Master_Pipeline/
│
├── Shared_Library/
│   ├── vars/
│   ├── src/
│   ├── resources/
│   └── docs/
│
├── Job_DSL/
│   ├── jobs/
│   └── README.md
│
├── Terraform/
│   ├── Modules/
│   ├── environments/
│   └── README.md
│
├── Ansible/
│   ├── roles/
│   ├── ci/
│   ├── cd/
│   ├── collections/
│   └── README.md
│
├── Packer/
│   ├── applications/
│   └── README.md
│
└── README.md
```

We will preserve the logical separation of Jenkins, Shared Library, Job DSL, Terraform, Ansible and Packer from the original project rather than turning everything into one undifferentiated script repository. 

---

# 5. OTMS CI/CD strategy

The pipeline will be built around reusable Shared Library functions.

### Application CI

Five application pipelines:

```text
Frontend CI
Employee CI
Attendance CI
Salary CI
Notification CI
```

Common quality/security stages:

```text
Checkout
   ↓
Gitleaks
   ↓
Formatting
   ↓
Syntax validation
   ↓
Dependencies
   ↓
Unit tests
   ↓
Trivy
   ↓
SonarQube
   ↓
DAST where appropriate
   ↓
Artifact
```

The original project uses Gitleaks, formatting/syntax checks, Trivy, SonarQube and ZAP as part of its CI strategy. 

---

# 6. Artifact strategy

This is important.

We will **not** simply build "latest".

Every artifact gets a traceable identity.

Example:

```text
Employee API

source SHA
    ↓
CI build #42
    ↓
employee-api artifact
    ↓
SHA256
    ↓
Packer
    ↓
AMI
```

We will retain provenance information such as:

```text
source commit
branch
CI build
artifact hash
Packer build
AMI ID
```

The original OTMS design already uses artifact manifests and release provenance; we'll preserve this principle. 

---

# 7. Packer strategy

Packer produces application AMIs.

Conceptually:

```text
Application artifact
       ↓
Packer
       ↓
Application AMI
       ↓
Terraform
       ↓
EC2
```

We will use **immutable versioned AMIs**.

For example:

```text
otms-employee-v1.0.0
otms-employee-v1.0.1
```

Not:

```text
otms-employee-latest
```

This makes rollback possible.

---

# 8. Terraform strategy

Terraform owns infrastructure.

```text
Terraform
   │
   ├── VPC
   ├── Subnets
   ├── Route tables
   ├── NAT/Internet routing as required
   ├── Security Groups
   ├── NACLs
   ├── ALB
   ├── Target Groups
   ├── ASGs
   ├── EC2
   ├── IAM
   └── supporting resources
```

The original project uses a modular Terraform structure including network, standalone VM/EC2 and autoscaling modules. 

### Important

We will **not copy the original AWS account ID, state bucket, private IPs, or other environment-specific values**.

Those belong to the original project, not your personal AWS environment. 

---

# 9. Ansible strategy

Ansible handles configuration management.

Initial DB layer:

```text
Ansible
 ├── PostgreSQL
 ├── Redis
 └── ScyllaDB
```

The original project uses Ansible over AWS SSM for private database hosts rather than requiring direct SSH access. 

We'll preserve that design where appropriate.

---

# 10. OTMS deployment lifecycle

The official deployment path will be:

```text
Jenkins
   ↓
CI
   ↓
Artifact
   ↓
Packer
   ↓
Terraform
   ↓
Ansible
   ↓
Application infrastructure
   ↓
Smoke tests
   ↓
SUCCESS
```

Rollback:

```text
Current AMI
     ↓
Previous known-good AMI
     ↓
Terraform
     ↓
EC2
```

Destroy:

```text
Jenkins
   ↓
Terraform Destroy
```

---

# 11. Repository #2 — OTMS-Docker

Now we take the **same application** and create a completely independent container deployment.

```text
Developer
    ↓
GitHub
    ↓
Jenkins
    ↓
CI
    ↓
Docker Build
    ↓
Security Scan
    ↓
ECR
    ↓
Docker Host
    ↓
OTMS
```

### Official deployment path

```text
Jenkins → Docker → ECR → Docker Host
```

Not manual Docker commands.

---

# 12. Docker strategy

Each service gets its own image:

```text
otms/frontend
otms/employee-api
otms/attendance-api
otms/salary-api
otms/notification-api
```

Images are versioned.

Example:

```text
employee-api:1.0.0
employee-api:1.0.1
```

And preferably promoted/identified by immutable digest.

---

# 13. Docker security

Each Dockerfile will be designed with:

* minimal base image
* non-root execution where practical
* no secrets baked into image
* `.dockerignore`
* health checks
* dependency scanning
* image vulnerability scanning
* deterministic/versioned builds

Pipeline:

```text
Checkout
 ↓
Gitleaks
 ↓
Tests
 ↓
SonarQube
 ↓
Docker Build
 ↓
Trivy Image Scan
 ↓
Push ECR
 ↓
Deploy
 ↓
Smoke Test
```

---

# 14. Docker deployment

For development we can support:

```text
docker compose
```

But:

> **Docker Compose is not our official production deployment mechanism.**

Official deployment remains:

```text
Jenkins
   ↓
Docker/ECR
   ↓
Docker Host
```

This preserves your "Jenkins-only deployment" requirement.

---

# 15. Docker rollback

Example:

```text
Current:

employee-api:1.0.2

Problem detected
       ↓
Rollback
       ↓
employee-api:1.0.1
```

Jenkins performs the rollback.

---

# 16. Repository #3 — OTMS-EKS

This is the Kubernetes evolution.

```text
Developer
    ↓
GitHub
    ↓
Jenkins
    ↓
CI
    ↓
Docker Build
    ↓
Trivy
    ↓
ECR
    ↓
Terraform
    ↓
EKS
    ↓
Kubernetes
    ↓
OTMS
```

---

# 17. EKS repository structure

```text
OTMS-EKS/
│
├── terraform/
│   └── eks/
│
├── k8s/
│   ├── namespace/
│   │
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ...
│   │
│   ├── employee/
│   ├── attendance/
│   ├── salary/
│   ├── notification/
│   │
│   ├── ingress/
│   ├── configmaps/
│   └── secrets/
│
├── docker/
│
├── Jenkinsfile
│
├── scripts/
│
└── README.md
```

---

# 18. Kubernetes architecture

```text
                    Internet
                       │
                       ▼
                    Ingress
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
      Frontend      APIs          APIs
       Service
          │
          ▼
        Pods
```

Five application deployments:

```text
frontend
employee-api
attendance-api
salary-api
notification-api
```

Each gets:

* Deployment
* Service
* health probes
* resource requests/limits
* configurable replicas

---

# 19. EKS infrastructure

Terraform will create/manage:

```text
VPC
Subnets
IAM
EKS Cluster
Node Groups
Security
Networking
Supporting AWS resources
```

Then Jenkins deploys Kubernetes resources.

---

# 20. EKS deployment pipeline

```text
Checkout
   ↓
Gitleaks
   ↓
Tests
   ↓
SonarQube
   ↓
Docker Build
   ↓
Trivy
   ↓
ECR Push
   ↓
Terraform Plan
   ↓
Approval
   ↓
Terraform Apply
   ↓
Kubernetes Deploy
   ↓
Rollout Status
   ↓
Smoke Test
```

Rollback:

```text
kubectl rollout undo
```

or an equivalent controlled Jenkins rollback operation.

The important thing is that **Jenkins performs the official operation**, rather than requiring you to manually run Kubernetes commands.

---

# 21. Repository #4 — OTMS-Monitoring

This is the clever part of the architecture.

Monitoring is **not part of OTMS, OTMS-Docker or OTMS-EKS**.

It is an independent platform.

```text
OTMS
   \
OTMS-Docker ───→ OTMS-Monitoring
   /
OTMS-EKS
```

It can monitor **whichever one is currently deployed**.

---

# 22. Monitoring modes

The Jenkins pipeline will have:

### ACTION

```text
MONITOR_ONLY
DEPLOY_AND_MONITOR
```

### DEPLOYMENT_TARGET

```text
OTMS
OTMS-Docker
OTMS-EKS
```

So:

```text
┌─────────────────────────────────────┐
│      OTMS Monitoring Pipeline       │
├─────────────────────────────────────┤
│                                     │
│ ACTION                              │
│  ○ MONITOR_ONLY                     │
│  ○ DEPLOY_AND_MONITOR               │
│                                     │
│ TARGET                              │
│  ○ OTMS                             │
│  ○ OTMS-Docker                      │
│  ○ OTMS-EKS                         │
│                                     │
└─────────────────────────────────────┘
```

---

# 23. Monitoring mode 1 — MONITOR_ONLY

Suppose:

```text
OTMS-Docker
```

is already running.

You run:

```text
OTMS-Monitoring
```

with:

```text
ACTION = MONITOR_ONLY
TARGET = OTMS-Docker
```

Pipeline:

```text
Validate target
       ↓
Discover infrastructure
       ↓
Configure monitoring
       ↓
Deploy monitoring
       ↓
Validate metrics
       ↓
SUCCESS
```

It **doesn't redeploy the application**.

---

# 24. Monitoring mode 2 — DEPLOY_AND_MONITOR

Suppose nothing is running.

You choose:

```text
ACTION = DEPLOY_AND_MONITOR
TARGET = OTMS-EKS
```

Then:

```text
OTMS-Monitoring
       ↓
Validate target
       ↓
Trigger OTMS-EKS Jenkins deployment
       ↓
Wait for successful deployment
       ↓
Validate application
       ↓
Deploy monitoring
       ↓
Configure monitoring
       ↓
Verify metrics
       ↓
SUCCESS
```

This makes `OTMS-Monitoring` a **deployment orchestrator**, but not the owner of the underlying deployment implementation.

---

# 25. Monitoring stack

Our planned monitoring stack:

```text
Prometheus
    ↓
Metrics collection
    ↓
Grafana
    ↓
Dashboards

Prometheus
    ↓
Alert rules
    ↓
Alertmanager
    ↓
Notifications
```

Potential exporters/components will depend on the deployment target.

### OTMS

Monitor:

```text
EC2
CPU
Memory
Disk
Network
Services
Application endpoints
```

### OTMS-Docker

Monitor:

```text
Docker Host
Containers
Container health
CPU
Memory
Network
Application endpoints
```

### OTMS-EKS

Monitor:

```text
Cluster
Nodes
Pods
Deployments
Services
Ingress
CPU
Memory
Application metrics
```

---

# 26. Monitoring should understand the target

The monitoring pipeline should not simply assume that every environment behaves identically.

Conceptually:

```text
TARGET=OTMS

        ↓

Use EC2 monitoring configuration
```

Whereas:

```text
TARGET=OTMS-Docker

        ↓

Use Docker monitoring configuration
```

And:

```text
TARGET=OTMS-EKS

        ↓

Use Kubernetes monitoring configuration
```

This is the main intelligence of the monitoring repository.

---

# 27. The four repositories must communicate through contracts

This is very important.

We don't want hardcoded assumptions scattered everywhere.

We'll establish contracts such as:

```text
Application version
Deployment target
Environment
Image tag
Image digest
AMI ID
Terraform outputs
Service endpoints
Monitoring endpoints
```

For example:

```text
OTMS-EKS
    ↓
publishes deployment information
    ↓
OTMS-Monitoring
    ↓
consumes deployment information
```

This could eventually use:

* Terraform outputs
* AWS tags
* SSM Parameter Store
* Secrets Manager
* ECR metadata
* Kubernetes labels
* configuration files

We'll decide the exact mechanism during implementation.

---

# 28. The most important independence rule

The relationship is:

```text
OTMS
   ↓
can run without Docker
can run without EKS
can run without Monitoring
```

```text
OTMS-Docker
   ↓
can run without OTMS EC2
can run without EKS
can run without Monitoring
```

```text
OTMS-EKS
   ↓
can run without OTMS EC2
can run without Docker Host
can run without Monitoring
```

```text
OTMS-Monitoring
   ↓
can run against any ONE selected target
```

This is our core architecture.

---

# 29. Direct deployment vs orchestrated deployment

We will deliberately support **both**.

## Direct

```text
OTMS
 ↓
Jenkins
 ↓
Deploy
```

or:

```text
OTMS-Docker
 ↓
Jenkins
 ↓
Deploy
```

or:

```text
OTMS-EKS
 ↓
Jenkins
 ↓
Deploy
```

Then separately:

```text
OTMS-Monitoring
 ↓
MONITOR_ONLY
 ↓
Attach monitoring
```

---

## Orchestrated

```text
OTMS-Monitoring
       ↓
TARGET = OTMS
       ↓
Trigger OTMS
       ↓
Wait
       ↓
Monitor OTMS
```

Or:

```text
OTMS-Monitoring
       ↓
TARGET = OTMS-Docker
       ↓
Trigger OTMS-Docker
       ↓
Wait
       ↓
Monitor Docker
```

Or:

```text
OTMS-Monitoring
       ↓
TARGET = OTMS-EKS
       ↓
Trigger OTMS-EKS
       ↓
Wait
       ↓
Monitor EKS
```

---

# 30. Security strategy across all four repositories

Security isn't a Phase 4 add-on.

It is built into each phase.

### Source security

```text
Gitleaks
```

### Code quality

```text
SonarQube
```

### Dependency/image security

```text
Trivy
```

### Secrets

Never:

```text
password=abc123
AWS_SECRET=...
DB_PASSWORD=...
```

inside Git.

Instead:

```text
Jenkins Credentials
AWS Secrets Manager
SSM Parameter Store
Kubernetes Secrets
```

depending on the deployment model.

---

# 31. Versioning strategy

Everything should be traceable.

Example:

```text
Application
v1.0.0

Docker
v1.0.0

AMI
v1.0.0

EKS
v1.0.0
```

And ideally:

```text
Git SHA
   ↓
CI Build
   ↓
Artifact
   ↓
Docker/AMI
   ↓
Deployment
   ↓
Monitoring
```

So we can answer:

> "Exactly which source code is running?"

without guessing.

---

# 32. Environment strategy

Initially:

```text
dev
```

But the design should allow:

```text
dev
staging
prod
```

without restructuring everything.

For example:

```text
OTMS
├── environments/
│   └── dev/
│
OTMS-Docker
├── environments/
│   └── dev/
│
OTMS-EKS
├── environments/
│   └── dev/
│
OTMS-Monitoring
└── environments/
    └── dev/
```

We can add staging/prod later.

---

# 33. Rollback strategy

Every deployment model gets its own rollback mechanism.

### OTMS

```text
Previous AMI
```

### OTMS-Docker

```text
Previous image tag/digest
```

### OTMS-EKS

```text
Previous Kubernetes revision/image
```

### Monitoring

Monitoring changes should also be versioned/configuration-controlled.

---

# 34. Destroy strategy

Every infrastructure-owning repository must have a safe destroy process.

```text
OTMS
    → Terraform Destroy

OTMS-Docker
    → Terraform/host destroy where applicable

OTMS-EKS
    → Terraform Destroy

OTMS-Monitoring
    → Monitoring infrastructure destroy
```

And we should eventually make the Jenkins jobs require explicit confirmation for destructive operations.

---

# 35. What we will NOT do

These are deliberate project rules.

### ❌ No Kubernetes in original OTMS

The original OTMS implementation is the VM/EC2-based version. 

### ❌ No Docker in original OTMS

Docker is introduced only in `OTMS-Docker`.

### ❌ No EKS in original OTMS

EKS is introduced only in `OTMS-EKS`.

### ❌ No monitoring embedded into every deployment repo

Monitoring belongs to `OTMS-Monitoring`.

### ❌ No three simultaneously required deployments

We can run any one of:

```text
OTMS
OTMS-Docker
OTMS-EKS
```

### ❌ No manual deployment as the official workflow

Jenkins is the deployment control plane.

---

# 36. Project evolution

The whole project tells one story:

```text
                    OTMS
                     │
             Traditional DevOps
                     │
          Jenkins + Terraform
          Ansible + Packer + EC2
                     │
                     ▼
                OTMS-Docker
                     │
                Containerization
                     │
             Docker + ECR + Jenkins
                     │
                     ▼
                 OTMS-EKS
                     │
                Orchestration
                     │
           Kubernetes + EKS + ECR
                     │
                     ▼
             OTMS-Monitoring
                     │
                Observability
                     │
        Prometheus + Grafana + Alerts
```

That's the narrative we will use throughout the project.

---

# 37. Final architecture

Here is the blueprint I want us to keep coming back to:

```text
                             GITHUB
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
             ▼                  ▼                  ▼
          OTMS            OTMS-Docker          OTMS-EKS
             │                  │                  │
             │                  │                  │
        Jenkins              Jenkins            Jenkins
             │                  │                  │
       ┌─────┼─────┐       ┌────┼────┐       ┌────┼─────┐
       │     │     │       │    │    │       │    │     │
     Packer TF  Ansible   Docker ECR Host   Docker ECR Terraform
       │     │     │                              │      │
       └─────┼─────┘                              │      │
             │                                    │      │
            EC2                                  EKS ◄──┘
             │                                    │
             └────────────────┬───────────────────┘
                              │
                              ▼
                     OTMS-Monitoring
                              │
                     ┌────────┴────────┐
                     │                 │
                MONITOR_ONLY     DEPLOY_AND_MONITOR
                     │                 │
                     ▼                 ▼
               Existing target    Selected target
                                     │
                            ┌────────┼────────┐
                            ▼        ▼        ▼
                           OTMS   Docker     EKS
```

---

# 38. Our implementation phases

Now our actual work plan becomes:

### PHASE 0 — Blueprint & Baseline

**Current phase**

* Freeze architecture
* Create four repositories
* Map uploaded source
* Identify original dependencies
* Identify environment-specific values
* Establish Git strategy

---

### PHASE 1 — OTMS

Build and verify:

```text
Application
↓
CI
↓
Shared Library
↓
Job DSL
↓
Packer
↓
Terraform
↓
Ansible
↓
EC2
↓
Smoke Test
↓
Rollback
↓
Destroy
```

**Do not move on until this independently works.**

---

### PHASE 2 — OTMS-Docker

Build and verify:

```text
Application
↓
Dockerfiles
↓
CI
↓
Docker Build
↓
Trivy
↓
ECR
↓
Docker Host
↓
Smoke Test
↓
Rollback
↓
Destroy
```

**Do not move on until this independently works.**

---

### PHASE 3 — OTMS-EKS

Build and verify:

```text
Application
↓
Docker
↓
ECR
↓
Terraform
↓
EKS
↓
Kubernetes
↓
Smoke Test
↓
Rollback
↓
Destroy
```

**Do not move on until this independently works.**

---

### PHASE 4 — OTMS-Monitoring

Build:

```text
Prometheus
Grafana
Alertmanager
Exporters
Dashboards
Alerts
```

Then support:

```text
MONITOR_ONLY
     +
OTMS
OTMS-Docker
OTMS-EKS
```

Then implement:

```text
DEPLOY_AND_MONITOR
```

where the monitoring repository can orchestrate the selected project's Jenkins pipeline.

---

### PHASE 5 — Final engineering review

This is **not** where we add basic production practices.

Those are implemented throughout Phases 1–4.

Phase 5 is where we test the entire solution:

```text
Security
Reliability
Rollback
Failure recovery
Observability
Cost
DR
Pipeline optimization
Documentation
Architecture
Interview readiness
```

---

# 39. Our "definition of done"

A phase isn't finished because "the application opened in the browser."

For each deployment model we need:

```text
[ ] Code builds
[ ] CI passes
[ ] Security scans pass
[ ] Infrastructure deploys
[ ] Application deploys
[ ] Health checks pass
[ ] Smoke tests pass
[ ] Logs available
[ ] Metrics available where applicable
[ ] Rollback tested
[ ] Destroy tested
[ ] Jenkins pipeline works
[ ] Secrets aren't committed
[ ] Documentation updated
```

And most importantly:

```text
[ ] Can deploy independently
[ ] Can destroy independently
[ ] Does not require another deployment model
```

---

# 40. The master rule for future chats

This is the part I especially recommend you keep.

Whenever we start a new chat, give me this:

> **"Continue the OTMS project from the Master Blueprint. We have four repositories: OTMS, OTMS-Docker, OTMS-EKS and OTMS-Monitoring. Each deployment model must be independently deployable through Jenkins. OTMS is the traditional Jenkins + Terraform + Ansible + Packer + EC2 implementation. OTMS-Docker is the independent Docker/ECR deployment. OTMS-EKS is the independent EKS/Kubernetes deployment. OTMS-Monitoring can either monitor an already-running selected deployment or orchestrate deployment of one selected deployment and then attach monitoring. Do not introduce Docker/Kubernetes into the original OTMS implementation. Work step-by-step, verify each step before proceeding, and use the uploaded OTMS Technical Knowledge Transfer and existing uploaded repositories as the baseline."**

Then add:

```text
CURRENT PHASE:
CURRENT REPOSITORY:
CURRENT STEP:
LAST SUCCESSFUL ACTION:
NEXT ACTION:
BLOCKERS:
```

For example:

```text
CURRENT PHASE: 0
CURRENT REPOSITORY: None
CURRENT STEP: Create repositories
LAST SUCCESSFUL ACTION: Blueprint finalized
NEXT ACTION: Create four empty GitHub repositories
BLOCKERS: None
```

That will make it **much easier to resume the project even in a completely new conversation**.

---
