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

- Azure subscription (free tier works for most exercises)
- GitHub account (free)
- Azure DevOps organization (free for up to 5 users)
- VS Code with Azure and GitHub extensions
- Azure CLI installed locally
- Terraform or Bicep CLI (for IaC exercises)

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
