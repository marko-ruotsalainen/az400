# Exercise 3.10: Pipeline Maintenance & Optimization

## Objective
Optimize pipelines for cost, performance, and reliability. Implement retention, caching, and migrate from classic to YAML.

## Skills Measured
- Monitor pipeline health, including failure rate, duration, and flaky tests
- Optimize a pipeline for cost, time, performance, and reliability
- Optimize pipeline concurrency for performance and cost
- Design and implement a retention strategy for pipeline artifacts and dependencies
- Migrate a pipeline from classic to YAML in Azure Pipelines

## Steps

### Part A: Pipeline Caching

```yaml
# npm caching
steps:
  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      path: '$(System.DefaultWorkingDirectory)/node_modules'
      cacheHitVar: 'CACHE_RESTORED'
    displayName: 'Cache npm packages'

  - script: npm ci
    condition: ne(variables.CACHE_RESTORED, 'true')
    displayName: 'npm install (cache miss)'
```

```yaml
# GitHub Actions caching
steps:
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-

  # Or use built-in cache in setup-node
  - uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: 'npm'
```

### Part B: Pipeline Concurrency

1. **Azure Pipelines parallel jobs**:
   ```yaml
   # Run tests in parallel with job splitting
   jobs:
     - job: Test_Unit
       pool:
         vmImage: 'ubuntu-latest'
       steps:
         - script: npm run test:unit

     - job: Test_Integration
       pool:
         vmImage: 'ubuntu-latest'
       steps:
         - script: npm run test:integration

     - job: Test_E2E
       pool:
         vmImage: 'ubuntu-latest'
       steps:
         - script: npm run test:e2e

     # Build only after ALL test jobs pass
     - job: Build
       dependsOn: [Test_Unit, Test_Integration, Test_E2E]
       steps:
         - script: npm run build
   ```

2. **GitHub Actions concurrency** (prevent duplicate runs):
   ```yaml
   concurrency:
     group: ${{ github.workflow }}-${{ github.ref }}
     cancel-in-progress: true  # Cancel previous run if new push
   ```

3. **Azure Pipelines exclusive lock**:
   ```yaml
   # Only one deployment at a time
   - stage: Deploy
     lockBehavior: sequential  # or 'runLatest'
     jobs:
       - deployment: Deploy
         environment: 'Production'
   ```

### Part C: Retention Policies

1. **Azure DevOps retention settings**:
   - Project Settings → Pipelines → Settings → Retention
   - Configure: Days to keep runs, minimum runs to keep
   - Per-pipeline overrides available

2. **Pipeline-level retention**:
   ```yaml
   # Keep production artifacts longer
   - stage: Deploy_Production
     jobs:
       - deployment: Deploy
         environment: 'Production'
         strategy:
           runOnce:
             deploy:
               steps:
                 - publish: $(Pipeline.Workspace)/drop
                   artifact: production-artifact
                   # Retention managed by environment settings
   ```

3. **GitHub Actions artifact retention**:
   ```yaml
   - uses: actions/upload-artifact@v4
     with:
       name: build-output
       path: dist/
       retention-days: 5  # Default is 90, reduce for cost
   ```

### Part D: Flaky Test Detection

```yaml
# Re-run failed tests to detect flaky behavior
steps:
  - script: |
      npm test || npm test  # Simple retry
    displayName: 'Run tests with retry'

  # Better: use test framework retry
  # Jest: --bail --forceExit
  # pytest: --reruns 2
```

In Azure DevOps, view Test Analytics:
- Pipelines → Test → Analytics
- Shows: Failure rate, flaky tests, test duration trends
- Action: Quarantine flaky tests, fix, or remove

### Part E: Pipeline Performance Optimization

1. **Use shallow clones** (fastest checkout):
   ```yaml
   # Azure Pipelines
   steps:
     - checkout: self
       fetchDepth: 1

   # GitHub Actions
   - uses: actions/checkout@v4
     with:
       fetch-depth: 1
   ```

2. **Use matrix strategy wisely** — only test what matters:
   ```yaml
   strategy:
     matrix:
       node18-linux:
         nodeVersion: '18'
         vmImage: 'ubuntu-latest'
       node20-linux:
         nodeVersion: '20'
         vmImage: 'ubuntu-latest'
     maxParallel: 2  # Limit parallel jobs to control cost
   ```

3. **Conditional stages** — skip unnecessary work:
   ```yaml
   - stage: Deploy
     condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
   ```

### Part F: Classic to YAML Migration

1. **Export classic pipeline**:
   - Open classic pipeline → View YAML (preview)
   - Copy the generated YAML as a starting point

2. **Key differences**:
   | Classic | YAML |
   |---------|------|
   | UI-based | Code-based |
   | Not version controlled | In source control |
   | Separate build/release | Unified multi-stage |
   | Task groups | Templates |
   | Variable groups (UI) | Variable groups + inline |

3. **Migration checklist**:
   - [ ] Export or document classic pipeline tasks
   - [ ] Create YAML file with equivalent stages/jobs/steps
   - [ ] Convert task groups to YAML templates
   - [ ] Move variables to variable groups or inline
   - [ ] Replicate triggers
   - [ ] Replicate approvals using YAML environments
   - [ ] Test in parallel with classic (don't delete classic until YAML is verified)
   - [ ] Disable classic pipeline

## Validation Checklist

- [ ] Pipeline caching implemented (Azure Pipelines and GitHub Actions)
- [ ] Parallel jobs configured for test splitting
- [ ] Concurrency controls configured
- [ ] Retention policies set for artifacts
- [ ] Flaky test detection understood
- [ ] Shallow clone optimization applied
- [ ] Classic to YAML migration steps understood

## Microsoft Learn References
- [Pipeline caching](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/caching)
- [Retention policies](https://learn.microsoft.com/en-us/azure/devops/pipelines/policies/retention)
- [Migrate from classic to YAML](https://learn.microsoft.com/en-us/azure/devops/pipelines/migrate/from-classic-pipelines)
