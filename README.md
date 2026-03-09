# AZ-400: Azure DevOps Engineer Expert — Hands-On Preparation

## Exam Breakdown & Study Time Allocation

| # | Domain | Exam Weight | Suggested Hours | Exercises |
|---|--------|-------------|-----------------|-----------|
| 1 | Design and implement processes and communications | 10–15% | 8–10 hrs | 6 |
| 2 | Design and implement a source control strategy | 10–15% | 8–10 hrs | 7 |
| 3 | Design and implement build and release pipelines | 50–55% | 35–40 hrs | 20 |
| 4 | Develop a security and compliance plan | 10–15% | 8–10 hrs | 7 |
| 5 | Implement an instrumentation strategy | 5–10% | 5–7 hrs | 5 |
| | **Total** | **100%** | **~65–75 hrs** | **45** |

## Preparation Strategy

1. **Hands-on labs (50%)** — Work through exercises in this repo
2. **Microsoft Learn paths (25%)** — Fill knowledge gaps with official docs
3. **Practice exams (15%)** — MeasureUp / Microsoft practice assessments
4. **Review & notes (10%)** — Summarize key concepts and revisit weak areas

## Prerequisites

### Accounts (Free)

- **Azure subscription** — [Create a free account](https://azure.microsoft.com/free/) with $200 credit. Most exercises work within the free tier.
- **GitHub account** — [Sign up free](https://github.com/signup). Some exercises use GitHub Advanced Security features, which require a paid plan or are free for public repos.
- **Azure DevOps organization** — [Create free at dev.azure.com](https://dev.azure.com). Free for up to 5 users, includes Azure Boards, Repos, Pipelines (1 free parallel job), and Artifacts (2 GB).

### CLI Tools (Install Locally)

These command-line tools are used throughout the exercises. Install them before starting.

| Tool | What It Is | Install | Used In |
|------|-----------|---------|--------|
| **Git** | Version control system | [git-scm.com](https://git-scm.com/downloads) | All exercises |
| **gh** (GitHub CLI) | Official command-line tool for GitHub. Lets you create repos, issues, PRs, labels, releases and more directly from the terminal instead of the GitHub web UI. | [cli.github.com](https://cli.github.com/) — then run `gh auth login` to authenticate | Domain 1, 2, 4 |
| **az** (Azure CLI) | Official command-line tool for managing Azure resources. Used to create/manage Azure services, query pipelines, deploy apps, and more. | [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) — then run `az login` to authenticate | Domain 1, 3, 4, 5 |
| **Node.js & npm** | JavaScript runtime and its package manager. `npm` installs project dependencies, runs scripts and tests. `npx` runs packages without installing them globally. | [nodejs.org](https://nodejs.org/) (LTS version, includes npm) | Domain 3, 4 |
| **Docker** | Container platform for building, running, and shipping applications as container images. | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) | Exercise 3.7, 3.9, 4.2 |
| **Terraform** | Infrastructure as Code tool by HashiCorp. Defines cloud resources in `.tf` files using HCL (HashiCorp Configuration Language). | [terraform.io/downloads](https://developer.hashicorp.com/terraform/install) | Exercise 3.8 |
| **Bicep CLI** | Azure-native IaC language (alternative to ARM/JSON templates). Bundled with Azure CLI, or install standalone. | Included with `az` CLI, or: `az bicep install` | Exercise 3.8 |

### Optional / Exercise-Specific Tools

These are only needed for specific exercises. Install them when you reach that exercise.

| Tool | What It Is | Install | Used In |
|------|-----------|---------|--------|
| **git-filter-repo** | Python tool for rewriting Git history (e.g., removing sensitive data from all commits). Replaces the older `git filter-branch`. | `pip install git-filter-repo` (requires Python 3) | Exercise 2.4 |
| **BFG Repo-Cleaner** | Simpler alternative to git-filter-repo for removing large files or secrets from Git history. | [rtyley.github.io/bfg-repo-cleaner](https://rtyley.github.io/bfg-repo-cleaner/) (requires Java) | Exercise 2.4 |
| **Scalar** | Microsoft tool for optimizing very large Git repositories (enables partial clone, sparse checkout, commit-graph). Included in Git 2.38+. | Bundled with Git 2.38+. Verify: `scalar version` | Exercise 2.4 |
| **Flyway** | Database migration tool. Applies versioned SQL migration scripts in order. | [flywaydb.org](https://flywaydb.org/download) | Exercise 3.9 |
| **kubectl** | Kubernetes CLI for deploying and managing containerized apps on AKS clusters. | [kubernetes.io/docs/tasks/tools](https://kubernetes.io/docs/tasks/tools/) or `az aks install-cli` | Exercise 3.7 |
| **Python 3 & pip** | Python runtime and package installer. Only needed for git-filter-repo installation. | [python.org](https://www.python.org/downloads/) | Exercise 2.4 |

### IDE

- **VS Code** with these extensions:
  - [Azure Account](https://marketplace.visualstudio.com/items?itemName=ms-vscode.azure-account)
  - [Azure Pipelines](https://marketplace.visualstudio.com/items?itemName=ms-azure-devops.azure-pipelines) (YAML syntax highlighting)
  - [GitHub Pull Requests](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)
  - [Bicep](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep) (for IaC exercises)
  - [HashiCorp Terraform](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform) (for IaC exercises)

### Quick Verification

Run these commands to verify your core tools are installed:

```bash
git --version          # Should be 2.38+ for Scalar support
gh --version           # GitHub CLI
az --version           # Azure CLI
node --version         # Node.js (20.x LTS recommended)
npm --version          # npm (comes with Node.js)
docker --version       # Docker (for container exercises)
terraform --version    # Terraform (for IaC exercises)
az bicep version       # Bicep (for IaC exercises)
```

## Repository Structure

```
az400/
├── README.md                          # This file
├── PROGRESS.md                        # Track your progress
├── hello.js                           # Sample app for pipeline exercises
├── package.json
├── cicd/
│   └── basic-pipeline.yaml            # Starter pipeline
├── exercises/
│   ├── 01-processes-and-communications/
│   ├── 02-source-control-strategy/
│   ├── 03-build-and-release-pipelines/
│   ├── 04-security-and-compliance/
│   └── 05-instrumentation-strategy/
└── solutions/                         # Reference solutions (try exercises first!)
```

## How To Use

1. Read each exercise's README
2. Follow the step-by-step instructions
3. Compare with reference solutions only after attempting
4. Mark completion in PROGRESS.md
5. Review concepts you struggled with in Microsoft Learn

## Key Resources

- [AZ-400 Learning Path](https://learn.microsoft.com/en-us/credentials/certifications/devops-engineer/)
- [Azure DevOps Documentation](https://learn.microsoft.com/en-us/azure/devops/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Azure Pipelines YAML Reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/)
- [Microsoft Practice Assessment](https://learn.microsoft.com/en-us/credentials/certifications/devops-engineer/practice/assessment?assessment-type=practice&assessmentId=56)
