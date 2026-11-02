# AI Scan Repos

> An AI-Powered DevSecOps Repository Analysis Platform

AI Scan Repos is an enterprise-oriented DevSecOps platform designed to analyze Git repositories by combining deterministic security scanners with AI-assisted reasoning.

The platform orchestrates multiple analysis engines, enriches their findings using Large Language Models (LLMs), and produces a single actionable report while ensuring customer source code is never permanently stored.

This repository acts as the documentation and architecture umbrella for the platform. The actual implementation is split across three active repositories:

- [ai-scanner-gateway](https://github.com/ai-scan-repo/ai-scanner-gateway) handles GitHub webhook ingestion, authentication, repository triggering, and orchestration of review requests.
- [ai-scanner-backend](https://github.com/ai-scan-repo/ai-scanner-backend) exposes the review API, manages review lifecycle state, resumes or restarts scans, and serves generated reports.
- [ai-scanner-frontend](https://github.com/ai-scan-repo/ai-scanner-frontend) provides the web interface for submitting repositories, monitoring progress, and reviewing findings.

Together, they form a security review platform that combines repository scanning, AI enrichment, evidence-based findings, and a clean operational workflow for teams reviewing code risk.

---

# Key Features

- Full repository security scanning
- Code quality analysis
- AI-assisted reasoning
- Summary and Deep Report modes
- Repository ownership verification
- Ephemeral Kubernetes Jobs
- Privacy-first architecture
- Plugin-based scanner framework
- AI provider abstraction
- Cloud-native deployment

---

# High-Level Architecture

The platform is composed of several independent services.

```text
Developer
      │
      ▼
Repository Provider
(GitHub / GitLab)
      │
      ▼
Backend API
      │
      ▼
Review Orchestrator
      │
      ▼
Ephemeral Kubernetes Job
      │
      ▼
Analysis Engine
      │
      ├── Semgrep
      ├── Gitleaks
      ├── Trivy
      ├── Quality Analysis
      └── AI Analysis
      │
      ▼
Unified Report
```

---

# Core Design Principles

- Security by Design
- Privacy by Default
- AI as an Augmentation Layer
- Full Repository Deterministic Scanning
- Ephemeral Repository Processing
- Trust-Based Authorization
- Modular Plugin Architecture
- AI Provider Independence
- Cloud Agnostic Deployment

---

# Installation Options and Repository Consolidation

There are two supported ways to deploy the platform. Choose one; the second
option automates much of the infrastructure and deployment work, but still
requires a few GitHub and cloud-provider actions from the operator.

## Option 1: Build and deploy it yourself

Use the [step-by-step installation guide](./Initial-the-project.md) to prepare
a Linux/K3s server, check out the application and deployment repositories,
build and import the images, configure credentials, and deploy the manifests.
This path gives you direct control over each command and the infrastructure.
You provision the cluster and supporting services yourself; the guide's
currently documented flow targets K3s, not Terraform-provisioned EKS.

The installation guide uses a GitHub owner placeholder in its clone examples.
Set it to the organization or account that owns your six application and
deployment repositories.

## Option 2: Use Elad's Docker installer image

The [installer repository and its README](https://github.com/ai-scan-repo/ai-scanner-quick-installer)
describe the wizard and provide the platform-specific `docker run` commands.
Use the installer's documented Docker Hub image name or its AWS ECR image URI,
depending on where you publish it. The wizard supports an existing local or
remote Kubernetes cluster and can provision EKS with Terraform. It reads the
three application and three deployment repositories, then writes generated
deployment environments and CI pipeline files to a branch in a selected
installer repository. It does not combine or replace the application source
repositories.

Before using this option, confirm that the image is available to you and that
the GitHub token can read the six repositories, manage webhooks on the three
application repositories, and push to the selected installer repository. EKS
also requires an AWS account and credentials. GitHub OAuth app creation and
some provider-side setup remain manual.

## Can `ai-scan-repo` and `ai-scanner-quick-installer` become one repository?

Yes. A single Git repository can retain the documentation repository's
contents at the root and place the installer files in their own subdirectory,
leaving all `ai-scanner-*` application and deployment repositories untouched.
Keeping the installer files together preserves its Docker build context and
keeps the documentation's existing relative paths intact.

There is one important configuration detail: the installer writes generated
manifests to a branch such as `env/local` or `env/eks` in its selected
installer repository. After consolidating these two repositories, select the
new combined repository in the wizard's **Installer repository** field and
grant the token push access to it. The installer default is intentionally
unchanged, so select the combined repository explicitly in the wizard rather
than relying on its default. The installer will continue to read the six
separate application/deployment repositories.

The current workspace contains the two repositories as separate directories,
not as an already-consolidated Git repository. This README documents the
options and required configuration; it does not perform a Git-history merge
or change any `ai-scanner-*` code.

---

# Documentation

Project documentation is located under the [docs](./docs/) directory.

| No. | Document | Link |
|---:|---|---|
| — | Complete Installation and Deployment Guide | [Initial-the-project.md](./Initial-the-project.md) |
| 01 | Executive Summary | [01-executive-summary.md](docs/01-executive-summary.md) |
| 02 | Business Requirements | [02-business-requirements.md](docs/02-business-requirements.md) |
| 03 | High-Level Design | [03-high-level-design/README.md](docs/03-high-level-design/README.md) |
| 04 | Low-Level Design | [04-low-level-design.md](docs/04-low-level-design.md) |
| 05 | Security and Authorization Design | [05-security-authorization-design.md](docs/05-security-authorization-design.md) |
| 06 | Threat Model | [06-threat-model.md](docs/06-threat-model.md) |
| 07 | API Specification | [07-api-specification.md](docs/07-api-specification.md) |
| 08 | Database Design | [08-database-design.md](docs/08-database-design.md) |
| 09 | Kubernetes Design | [09-kubernetes-design.md](docs/09-kubernetes-design.md) |
| 10 | Deployment Guide | [10-deployment-guide.md](docs/10-deployment-guide.md) |
| 11 | Monitoring and Observability | [11-monitoring-observability.md](docs/11-monitoring-observability.md) |
| 12 | Roadmap | [12-roadmap.md](docs/12-roadmap.md) |
| 13 | Future Enhancements | [13-future-enhancements.md](docs/13-future-enhancements.md) |
| 14 | MCP Adaptation Plan | [14-mcp-adaptation-plan.md](docs/14-mcp-adaptation-plan.md) |
| 15 | MCP Server Documentation | [15-mcp-server-documentation.md](docs/15-mcp-server-documentation.md) |
| 16 | Gemma 4 26B User Guide | [16-gemma-user-guide.md](docs/16-gemma-user-guide.md) |
| 17 | Review API (Flask) | [17-review-api.md](docs/17-review-api.md) |
| 18 | Team Setup Guide | [18-team-setup-guide.md](docs/18-team-setup-guide.md) |
| 19 | Secret Exposure — Limits Analysis | [19-secret-exposure-limits.md](docs/19-secret-exposure-limits.md) |
| 20 | Phase 3 Plan | [20-phase-3-plan.md](docs/20-phase-3-plan.md) |
| 21 | Writing the Kubernetes manifests | [21-kubernetes-manifests-guide.md](docs/21-kubernetes-manifests-guide.md) |

---

# Project Status

A K3s deployment was documented as working on 2026-09-30 (namespace ai-scan-repos); see [DEPLOY-K3S.md](https://github.com/ai-scan-repo/ai-scanner-backend-deployment/blob/main/DEPLOY-K3S.md). The elad/ui-redesign-pause-resume branch was merged into main via PR #10 on 2026-10-02.

What is done and what is next: [docs/WORK-TRACKER.md](./docs/WORK-TRACKER.md). Where a new session
picks up: [HANDOFF.md](./HANDOFF.md). Notes for Claude Code: [CLAUDE.md](./CLAUDE.md).

---

# Repository Structure

```text
ai-scan-repo/
├── README.md
├── Initial-the-project.md
├── docs/
│   ├── index.md
│   ├── 01-executive-summary.md
│   ├── 02-business-requirements.md
│   ├── 03-high-level-design/
│   │   └── README.md
│   ├── 04-low-level-design.md
│   ├── 05-security-authorization-design.md
│   ├── 06-threat-model.md
│   ├── 07-api-specification.md
│   ├── 08-database-design.md
│   ├── 09-kubernetes-design.md
│   ├── 10-deployment-guide.md
│   ├── 11-monitoring-observability.md
│   ├── 12-roadmap.md
│   ├── 13-future-enhancements.md
│   ├── 14-mcp-adaptation-plan.md
│   ├── 15-mcp-server-documentation.md
│   ├── 16-gemma-user-guide.md
│   ├── 17-review-api.md
│   ├── 18-team-setup-guide.md
│   ├── 19-secret-exposure-limits.md
│   ├── 20-phase-3-plan.md
│   ├── 21-kubernetes-manifests-guide.md
│   ├── adr/
│   └── diagrams/
├── reports/


├── ai-scanner-backend/
│   ├── README.md
│   ├── app/
│   ├── tests/
│   ├── Dockerfile
│   └── ...
├── ai-scanner-frontend/
│   ├── README.md
│   ├── src/
│   ├── package.json
│   └── ...
├── ai-scanner-gateway/
│   ├── README.md
│   ├── src/
│   ├── tests/
│   └── ...
```

---

# License

This project is currently under active development.
License information will be added before the first public release.
