# DevSecOps CI/CD Pipeline on AWS and Kubernetes

A complete DevSecOps setup that takes a Java (Spring Boot) web app from a Git commit to a running Kubernetes deployment. Security scanning and quality checks run at every stage, and the infrastructure and the app are both monitored.

The infrastructure is built with **Terraform** on AWS. The pipeline runs in **Jenkins**: it builds with **Maven**, checks code quality with **SonarQube**, scans for vulnerabilities with **Trivy**, stores build artifacts in **Nexus**, builds and pushes images to **Docker Hub**, and deploys to a self-managed **kubeadm** cluster. **Prometheus** and **Grafana** handle monitoring.

![Terraform](https://img.shields.io/badge/Terraform-7B42BC?logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?logo=amazonwebservices&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?logo=jenkins&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?logo=aqua&logoColor=white)
![Nexus](https://img.shields.io/badge/Nexus-1B1C30?logo=sonatype&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)

---

## Architecture

### Infrastructure (AWS, `us-east-2`)

Terraform provisions seven Ubuntu 24.04 EC2 instances in the default VPC. Each instance is set up on first boot by a `user_data` script.

```mermaid
flowchart TB
    dev([Developer]) -->|git push| git[(Private Git repo)]
    tf([Terraform]) -->|provisions| aws
    tf -.->|remote state| s3[(S3 backend<br/>native state locking)]

    subgraph aws[AWS · default VPC · us-east-2]
        direction TB

        subgraph ci[CI/CD toolchain]
            jenkins[Jenkins VM<br/>Jenkins · JDK 17 · Docker<br/>Trivy · kubectl · node_exporter]
            sonar[SonarQube VM<br/>sonarqube:lts-community · :9000]
            nexus[Nexus VM<br/>sonatype/nexus3 · :8081]
        end

        subgraph k8s[Kubernetes cluster · kubeadm v1.32 · CRI-O · Calico]
            master[k8s-master<br/>control plane]
            w1[k8s-worker-1]
            w2[k8s-worker-2]
            master --- w1
            master --- w2
        end

        subgraph mon[Monitoring VM]
            prom[Prometheus · :9090]
            graf[Grafana · :3000]
            bb[Blackbox Exporter · :9115]
        end
    end

    git -->|webhook / poll| jenkins
    jenkins <-->|analysis + quality gate webhook| sonar
    jenkins -->|publish JAR| nexus
    jenkins -->|push image| hub[(Docker Hub)]
    jenkins -->|kubectl apply<br/>ServiceAccount token| master
    hub -->|pull image| w1
    hub -->|pull image| w2

    graf --> prom
    prom --> bb
    bb -.->|HTTP probe| k8s
    prom -.->|scrape /prometheus + :9100| jenkins
    jenkins -.->|email notification| mail([Gmail SMTP])
```

### CI/CD pipeline flow

```mermaid
flowchart LR
    A[Checkout] --> B[Compile<br/>Maven]
    B --> C[Unit tests<br/>Maven]
    C --> D[Filesystem scan<br/>Trivy]
    D --> E[Code analysis<br/>SonarQube]
    E --> F{Quality gate}
    F -->|fail| X([Stop + email])
    F -->|pass| G[Package<br/>Maven]
    G --> H[Publish artifact<br/>Nexus]
    H --> I[Build + tag image<br/>Docker]
    I --> J[Image scan<br/>Trivy]
    J --> K[Push image<br/>Docker Hub]
    K --> L[Deploy<br/>Kubernetes]
    L --> M[Verify rollout]
    M --> N([Email report])
```

Security runs through the whole pipeline: Trivy scans both the source tree and the built image, SonarQube's quality gate stops bad code before anything is packaged, and Jenkins deploys through a namespace-scoped ServiceAccount instead of cluster-admin credentials.

---

## Tech stack

| Area | Tools |
|---|---|
| Infrastructure as code | Terraform ≥ 1.10.5, AWS provider 5.85, S3 remote state with native locking |
| Cloud | AWS EC2, security groups, default VPC |
| Container orchestration | Kubernetes v1.32 (kubeadm), CRI-O, Calico CNI, ingress-nginx |
| CI/CD | Jenkins, Maven, JDK 17 (Eclipse Temurin) |
| Code quality and security | SonarQube (SAST and quality gate), Trivy (filesystem and image scanning) |
| Artifacts and registry | Sonatype Nexus 3 (Maven releases and snapshots), Docker Hub |
| Monitoring | Prometheus 3.2, Grafana 11.5, Blackbox Exporter, Node Exporter, Jenkins Prometheus plugin |
| Notifications | Jenkins Extended E-mail via Gmail SMTP |

---

## Repository layout

```
.
├── phase-1-infra/            # Terraform: EC2 instances, security groups, S3 backend
│   ├── extras/               # user_data scripts (K8s nodes, Jenkins, Nexus/SonarQube, monitoring)
│   └── PHASE-1.md            # cluster bootstrap and tool setup guide
├── phase-2-git/              # Private repository setup for the application
├── phase-3-cicd/             # Jenkins configuration and Kubernetes manifests
│   ├── sa.yml                # Jenkins ServiceAccount (namespace: webapps)
│   ├── role.yml              # Role scoped to the webapps namespace
│   ├── role-binding.yml      # Binds the Role to the Jenkins ServiceAccount
│   ├── secret-token.yml      # Long-lived ServiceAccount token for Jenkins
│   └── deployment-service.yml# App Deployment (2 replicas) and Service
└── phase-4-monitoring/       # Prometheus, Grafana and exporter setup guide
```

---

## Project phases

| Phase | What it covers | Guide |
|---|---|---|
| 1. Infrastructure | Terraform provisioning, kubeadm cluster bootstrap, Calico, ingress-nginx, SonarQube and Nexus containers, Jenkins install | [PHASE-1.md](phase-1-infra/PHASE-1.md) |
| 2. Source control | Private application repository | [PHASE-2.md](phase-2-git/PHASE-2.md) |
| 3. CI/CD | Jenkins plugins and tools, Trivy, SonarQube webhook, Nexus `settings.xml`, Kubernetes RBAC for Jenkins, email alerts | [PHASE-3.md](phase-3-cicd/PHASE-3.md) |
| 4. Monitoring | Prometheus service, Grafana dashboards (Blackbox 7587, Node Exporter 1860, Jenkins 9964) | [PHASE-4.md](phase-4-monitoring/PHASE-4.md) |

---

## Quick start

**Prerequisites:** an AWS account with credentials configured, Terraform ≥ 1.10.5, an S3 bucket for remote state, and an EC2 key pair named `DevSecOpProjectKeyPair` in `us-east-2`.

```bash
git clone https://github.com/Gulnazzholshy01/DevSecOpProject.git
cd DevSecOpProject/phase-1-infra

# Update backend.tf with your own S3 bucket, and providers.tf with your AWS profile
terraform init
terraform plan
terraform apply
```

Then follow the phase guides in order, starting with [PHASE-1.md](phase-1-infra/PHASE-1.md), to bootstrap the cluster and configure each tool.

To tear everything down:

```bash
terraform destroy
```

> **Cost note:** this creates seven `t2.medium` instances, which are not covered by the AWS free tier. Destroy the environment when you're done.

---

## Author

**Gulnaz Zholshy** · [GitHub](https://github.com/Gulnazzholshy01)
