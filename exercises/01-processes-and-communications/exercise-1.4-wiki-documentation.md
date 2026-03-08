# Exercise 1.4: Wiki, Documentation & Mermaid Diagrams

## Objective
Document a project using wikis, Markdown, and Mermaid syntax. Automate release note generation from Git history.

## Skills Measured
- Document a project by configuring wikis and process diagrams, including Markdown and Mermaid syntax
- Configure release documentation, including release notes and API documentation
- Automate creation of documentation from Git history

## Steps

### Part A: Create a Project Wiki in Azure DevOps

1. **Create a Wiki** in Azure DevOps:
   - Overview → Wiki → Create project wiki
   - Create pages:
     - Home (project overview)
     - Architecture
     - Getting Started
     - API Reference
     - Runbooks

2. **Write Architecture page** using Mermaid diagrams:

   ```markdown
   # Architecture

   ## System Overview

   ```mermaid
   graph TD
       A[User] -->|HTTP Request| B[Load Balancer]
       B --> C[App Service - Slot 1]
       B --> D[App Service - Slot 2]
       C --> E[Azure SQL Database]
       D --> E
       C --> F[Azure Cache for Redis]
       D --> F
       E --> G[Azure Backup]
   ```

   ## CI/CD Flow

   ```mermaid
   flowchart LR
       A[Developer] -->|Push| B[GitHub]
       B -->|Trigger| C[GitHub Actions / Azure Pipelines]
       C -->|Build| D[Container Registry]
       C -->|Test| E[Test Results]
       D -->|Deploy| F[Dev Environment]
       F -->|Approval| G[Staging]
       G -->|Approval| H[Production]
   ```

   ## Sequence Diagram - Deployment

   ```mermaid
   sequenceDiagram
       participant Dev as Developer
       participant GH as GitHub
       participant CI as CI Pipeline
       participant ACR as Container Registry
       participant AKS as Kubernetes

       Dev->>GH: Push to main
       GH->>CI: Trigger pipeline
       CI->>CI: Build & Test
       CI->>ACR: Push image
       CI->>AKS: Deploy to staging
       AKS-->>CI: Health check passed
       CI->>AKS: Swap to production
   ```
   ```

3. **Publish as code wiki**:
   - You can also publish a folder in your repo as a wiki
   - Repos → Wiki → Publish code as wiki
   - Select the `/docs` folder

### Part B: Create Mermaid Diagrams in GitHub

1. **Create a docs folder** in your repo and add architecture docs:

   Create `docs/architecture.md` with the Mermaid diagrams above.
   GitHub natively renders Mermaid in Markdown files.

2. **Create a state diagram** for your work item flow:
   ```mermaid
   stateDiagram-v2
       [*] --> New
       New --> Active: Start Work
       Active --> Resolved: Code Complete
       Resolved --> Closed: Verified
       Resolved --> Active: Reopened
       Closed --> [*]
   ```

3. **Create a Gantt chart** for sprint planning:
   ```mermaid
   gantt
       title Sprint 1 Plan
       dateFormat  YYYY-MM-DD
       section Backend
       API endpoints      :a1, 2026-03-09, 5d
       Database migration  :a2, after a1, 3d
       section Frontend
       UI components      :b1, 2026-03-09, 7d
       section Testing
       Integration tests  :c1, after a2, 3d
       Load testing       :c2, after c1, 2d
   ```

### Part C: Automate Release Notes from Git History

1. **Use conventional commits** for structured history:
   ```
   feat: add user authentication
   fix: resolve timeout issue on health endpoint
   docs: update API reference
   chore: upgrade dependencies
   BREAKING CHANGE: remove deprecated v1 API
   ```

2. **Create a script** to generate release notes (`scripts/generate-release-notes.sh`):
   ```bash
   #!/bin/bash
   # Generate release notes from Git history between two tags
   PREVIOUS_TAG=$1
   CURRENT_TAG=$2

   if [ -z "$PREVIOUS_TAG" ] || [ -z "$CURRENT_TAG" ]; then
     echo "Usage: ./generate-release-notes.sh <previous-tag> <current-tag>"
     exit 1
   fi

   echo "# Release Notes: $CURRENT_TAG"
   echo ""
   echo "## Changes since $PREVIOUS_TAG"
   echo ""

   echo "### Features"
   git log "$PREVIOUS_TAG".."$CURRENT_TAG" --pretty=format:"- %s (%h)" --grep="^feat"
   echo ""

   echo "### Bug Fixes"
   git log "$PREVIOUS_TAG".."$CURRENT_TAG" --pretty=format:"- %s (%h)" --grep="^fix"
   echo ""

   echo "### Other Changes"
   git log "$PREVIOUS_TAG".."$CURRENT_TAG" --pretty=format:"- %s (%h)" --grep="^chore\|^docs\|^refactor"
   echo ""

   echo "### Contributors"
   git log "$PREVIOUS_TAG".."$CURRENT_TAG" --pretty=format:"%an" | sort -u | while read author; do
     echo "- $author"
   done
   ```

3. **Create a GitHub Actions workflow** for automated release notes:
   ```yaml
   # .github/workflows/release-notes.yml
   name: Generate Release Notes
   on:
     release:
       types: [created]

   jobs:
     release-notes:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0  # Full history for git log

         - name: Generate Release Notes
           uses: actions/github-script@v7
           with:
             script: |
               const { data: releases } = await github.rest.repos.listReleases({
                 owner: context.repo.owner,
                 repo: context.repo.repo,
               });
               const currentTag = context.payload.release.tag_name;
               const previousTag = releases.length > 1 ? releases[1].tag_name : '';

               const { data: comparison } = await github.rest.repos.compareCommits({
                 owner: context.repo.owner,
                 repo: context.repo.repo,
                 base: previousTag || context.payload.release.target_commitish,
                 head: currentTag,
               });

               let notes = `## What's Changed\n\n`;
               for (const commit of comparison.commits) {
                 notes += `- ${commit.commit.message.split('\n')[0]} (${commit.sha.substring(0, 7)})\n`;
               }

               await github.rest.repos.updateRelease({
                 owner: context.repo.owner,
                 repo: context.repo.repo,
                 release_id: context.payload.release.id,
                 body: notes,
               });
   ```

### Part D: Webhooks for Documentation Events

1. **Configure a webhook** in GitHub:
   - Settings → Webhooks → Add webhook
   - Configure for `release` and `wiki` events
   - Point to a service or Azure Function that processes updates

## Validation Checklist

- [ ] Azure DevOps wiki created with multiple pages
- [ ] Mermaid diagrams render correctly (flowchart, sequence, state, gantt)
- [ ] Code-as-wiki configured from a repository folder
- [ ] Conventional commit messages used in the repository
- [ ] Release notes generation script works
- [ ] GitHub Actions release notes workflow created
- [ ] Understand webhook configuration

## Key Concepts to Review

- **Mermaid diagram types**: flowchart, sequence, state, gantt, class, ER
- **Code wiki vs project wiki**: Code wiki is versioned in the repo
- **Conventional Commits**: Structured commit format for automation
- **GitHub Releases**: Tag-based release management with notes

## Microsoft Learn References
- [About wikis, READMEs, and Markdown](https://learn.microsoft.com/en-us/azure/devops/project/wiki/about-readme-wiki)
- [Mermaid syntax in Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/project/wiki/wiki-markdown-guidance?#add-mermaid-diagrams-to-a-wiki-page)
