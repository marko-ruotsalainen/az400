# Exercise 3.3: GitHub Actions Fundamentals

## Objective
Build CI/CD workflows using GitHub Actions with various triggers, job configurations, and runner types.

## Skills Measured
- Select a deployment automation solution, including GitHub Actions
- Develop and implement pipeline trigger rules
- Design and implement a strategy for job execution order, including parallelism and multi-stage pipelines

## Prerequisites
- **Node.js & npm** — JavaScript runtime and package manager. The sample app uses npm for dependencies and test scripts.
  - Install: https://nodejs.org/ (LTS version)
  - Verify: `node --version && npm --version`
- **Git** installed (`git --version` to verify)
- GitHub account with a repository

## Steps

### Part A: Basic Workflow Structure

1. **Create a CI workflow** (`.github/workflows/ci.yml`):
   ```yaml
   name: CI Pipeline

   on:
     push:
       branches: [main, develop]
       paths-ignore:
         - '*.md'
         - 'docs/**'
     pull_request:
       branches: [main]
     workflow_dispatch:  # Manual trigger
       inputs:
         environment:
           description: 'Target environment'
           required: true
           default: 'dev'
           type: choice
           options: [dev, staging, production]

   env:
     NODE_VERSION: '20'

   jobs:
     lint:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: ${{ env.NODE_VERSION }}
             cache: 'npm'
         - run: npm ci
         - run: npm run lint

     test:
       runs-on: ubuntu-latest
       needs: lint  # Depends on lint job
       strategy:
         matrix:
           node-version: [18, 20, 22]
           os: [ubuntu-latest, windows-latest]
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: ${{ matrix.node-version }}
         - run: npm ci
         - run: npm test

     build:
       runs-on: ubuntu-latest
       needs: test  # Depends on test job
       steps:
         - uses: actions/checkout@v4
         - uses: actions/setup-node@v4
           with:
             node-version: ${{ env.NODE_VERSION }}
             cache: 'npm'
         - run: npm ci
         - run: npm run build

         - uses: actions/upload-artifact@v4
           with:
             name: build-output
             path: dist/
             retention-days: 5

     deploy-dev:
       runs-on: ubuntu-latest
       needs: build
       if: github.ref == 'refs/heads/main'
       environment: development
       steps:
         - uses: actions/download-artifact@v4
           with:
             name: build-output
         - run: echo "Deploying to dev..."
   ```

### Part B: Trigger Types

Study and practice all trigger types:

```yaml
on:
  # Push trigger
  push:
    branches: [main, 'release/**']
    tags: ['v*']
    paths: ['src/**']
    paths-ignore: ['docs/**']

  # PR trigger
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  # Schedule (cron)
  schedule:
    - cron: '0 6 * * 1-5'  # Weekdays at 6 AM UTC

  # Manual
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy'
        required: true
        type: string

  # After another workflow
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
    branches: [main]

  # On release
  release:
    types: [published]

  # On issue/PR comment
  issue_comment:
    types: [created]
```

### Part C: Job Dependencies and Parallelism

```yaml
jobs:
  # These run in PARALLEL
  lint:
    runs-on: ubuntu-latest
    steps: [...]

  security-scan:
    runs-on: ubuntu-latest
    steps: [...]

  # This waits for BOTH lint and security-scan
  build:
    needs: [lint, security-scan]
    runs-on: ubuntu-latest
    steps: [...]

  # These run in PARALLEL after build
  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps: [...]

  integration-tests:
    needs: build
    runs-on: ubuntu-latest
    steps: [...]

  # This waits for BOTH staging deploy and integration tests
  deploy-production:
    needs: [deploy-staging, integration-tests]
    runs-on: ubuntu-latest
    environment: production
    steps: [...]
```

### Part D: Contexts and Expressions

```yaml
jobs:
  example:
    runs-on: ubuntu-latest
    steps:
      # Conditional execution
      - name: Only on main
        if: github.ref == 'refs/heads/main'
        run: echo "On main branch"

      - name: Only on PR
        if: github.event_name == 'pull_request'
        run: echo "This is a PR"

      - name: Always run (even on failure)
        if: always()
        run: echo "Cleanup"

      - name: Only on failure
        if: failure()
        run: echo "Something failed"

      - name: Skip on fork
        if: github.repository == 'owner/repo'
        run: echo "Not a fork"

      # Using expressions
      - name: Conditional with contains
        if: contains(github.event.head_commit.message, '[skip ci]') != true
        run: echo "Running CI"
```

### Part E: Secrets and Variables

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      # Repository variable
      APP_NAME: ${{ vars.APP_NAME }}
    steps:
      # Using secrets
      - name: Login to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      # Using environment variables
      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Deploying ${{ vars.APP_NAME }} to ${{ vars.ENVIRONMENT }}"
          # Never echo secrets!
```

### Part F: Reusable Workflows

1. **Create a reusable workflow** (`.github/workflows/reusable-deploy.yml`):
   ```yaml
   name: Reusable Deployment
   on:
     workflow_call:
       inputs:
         environment:
           required: true
           type: string
         artifact-name:
           required: true
           type: string
       secrets:
         azure-credentials:
           required: true

   jobs:
     deploy:
       runs-on: ubuntu-latest
       environment: ${{ inputs.environment }}
       steps:
         - uses: actions/download-artifact@v4
           with:
             name: ${{ inputs.artifact-name }}
         - uses: azure/login@v2
           with:
             creds: ${{ secrets.azure-credentials }}
         - run: echo "Deploying to ${{ inputs.environment }}"
   ```

2. **Call the reusable workflow**:
   ```yaml
   jobs:
     deploy-staging:
       uses: ./.github/workflows/reusable-deploy.yml
       with:
         environment: staging
         artifact-name: build-output
       secrets:
         azure-credentials: ${{ secrets.AZURE_CREDENTIALS }}
   ```

## Validation Checklist

- [ ] CI workflow created with lint → test → build → deploy stages
- [ ] All trigger types understood (push, PR, schedule, manual, workflow_run)
- [ ] Matrix strategy implemented for multi-version/multi-OS testing
- [ ] Job dependencies and parallelism configured
- [ ] Conditional execution with `if` expressions practiced
- [ ] Secrets and variables used correctly
- [ ] Reusable workflow created and called
- [ ] Artifacts uploaded and downloaded between jobs

## Microsoft Learn References
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Workflow syntax](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [Reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
