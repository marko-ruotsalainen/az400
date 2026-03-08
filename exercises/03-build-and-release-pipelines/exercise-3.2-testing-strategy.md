# Exercise 3.2: Testing Strategy in Pipelines

## Objective
Implement a comprehensive testing strategy with quality gates, multiple test types, and code coverage analysis.

## Skills Measured
- Design and implement quality and release gates, including security and governance
- Design a comprehensive testing strategy (local, unit, integration, load tests)
- Implement tests in a pipeline, including configuring test tasks and integration of test results
- Implement code coverage analysis

## Steps

### Part A: Add Tests to the Sample App

1. **Install test dependencies**:
   ```bash
   cd /home/marko/source/az400
   npm install --save-dev jest supertest
   ```

2. **Update `package.json`** with test script:
   ```json
   {
     "scripts": {
       "start": "node hello.js",
       "test": "jest --coverage --ci --reporters=default --reporters=jest-junit"
     },
     "devDependencies": {
       "jest": "^29.0.0",
       "supertest": "^6.0.0",
       "jest-junit": "^16.0.0"
     },
     "jest": {
       "coverageReporters": ["text", "cobertura", "lcov"],
       "coverageThreshold": {
         "global": {
           "branches": 80,
           "functions": 80,
           "lines": 80,
           "statements": 80
         }
       }
     }
   }
   ```

3. **Create unit tests** (`hello.test.js`):
   ```javascript
   const http = require('http');

   describe('Hello World Server', () => {
     let server;

     beforeAll((done) => {
       server = require('./hello');
       // Wait for server to start
       setTimeout(done, 100);
     });

     afterAll((done) => {
       server.close(done);
     });

     test('returns 200 for root path', (done) => {
       http.get('http://127.0.0.1:3000/', (res) => {
         expect(res.statusCode).toBe(200);
         done();
       });
     });

     test('returns Hello World', (done) => {
       http.get('http://127.0.0.1:3000/', (res) => {
         let data = '';
         res.on('data', (chunk) => (data += chunk));
         res.on('end', () => {
           expect(data).toContain('Hello, World!');
           done();
         });
       });
     });
   });
   ```

### Part B: Azure Pipelines — Testing with Code Coverage

```yaml
# cicd/pipeline-with-tests.yaml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '20.x'

          - script: npm install
            displayName: 'Install dependencies'

          - script: npm test
            displayName: 'Run tests with coverage'
            env:
              JEST_JUNIT_OUTPUT_DIR: $(System.DefaultWorkingDirectory)/test-results

          # Publish test results
          - task: PublishTestResults@2
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/junit.xml'
              mergeTestResults: true
              testRunTitle: 'Unit Tests'
            condition: always()

          # Publish code coverage
          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'
            condition: always()
```

### Part C: GitHub Actions — Testing with Code Coverage

```yaml
# .github/workflows/test.yml
name: Test and Coverage
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm install
      - run: npm test

      # Upload coverage report
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

      # Add coverage comment to PR
      - name: Coverage Report
        if: github.event_name == 'pull_request'
        uses: davelosert/vitest-coverage-report-action@v2
        with:
          json-summary-path: coverage/coverage-summary.json
```

### Part D: Quality Gates and Release Gates

1. **Build validation as a quality gate** (Azure DevOps):
   - Branch policies → Build validation
   - Required: Tests must pass before PR merge
   - Code coverage threshold: 80%

2. **Azure DevOps Release Gates**:
   ```yaml
   # YAML environment with checks
   stages:
     - stage: Deploy_Staging
       jobs:
         - deployment: DeployStaging
           environment: 'Staging'  # Configure checks on this environment
           strategy:
             runOnce:
               deploy:
                 steps:
                   - script: echo "Deploying to staging"

     - stage: Deploy_Production
       dependsOn: Deploy_Staging
       jobs:
         - deployment: DeployProd
           environment: 'Production'  # Has approval gate
           strategy:
             runOnce:
               deploy:
                 steps:
                   - script: echo "Deploying to production"
   ```

3. **Configure environment approvals** in Azure DevOps:
   - Pipelines → Environments → Production → Approvals and checks
   - Add: Approvals (require specific users/groups)
   - Add: Business hours (only deploy Mon-Fri 9am-5pm)
   - Add: Invoke Azure Function (custom gate logic)
   - Add: Query Azure Monitor alerts (no active incidents)

4. **GitHub environment protection rules**:
   - Settings → Environments → Production
   - Required reviewers: Add approvers
   - Wait timer: 30 minutes
   - Deployment branches: Only `main`

### Part E: Testing Pyramid

Study and implement the testing pyramid:

```
         /\
        /  \       E2E Tests (few, slow, expensive)
       / E2E\      - Selenium, Playwright, Cypress
      /------\
     /  Int.  \    Integration Tests (moderate)
    /  Tests   \   - API tests, database tests
   /------------\
  / Unit Tests   \ Unit Tests (many, fast, cheap)
 /________________\ - Jest, NUnit, PyTest
```

| Test Type | Speed | Quantity | Cost | When to Run |
|-----------|-------|----------|------|-------------|
| Unit | Fast | Many | Low | Every commit |
| Integration | Medium | Moderate | Medium | Every PR |
| E2E | Slow | Few | High | Pre-deploy |
| Load | Slow | Few | High | Pre-release |

### Part F: Load Testing

```yaml
# Azure Load Testing in pipeline
steps:
  - task: AzureLoadTest@1
    inputs:
      azureSubscription: 'Azure-Connection'
      loadTestConfigFile: 'loadtest/config.yaml'
      loadTestResource: 'my-load-test-resource'
      resourceGroup: 'rg-loadtest'
    displayName: 'Run load test'
```

## Validation Checklist

- [ ] Unit tests created and running locally
- [ ] Tests integrated into Azure Pipeline with published results
- [ ] Tests integrated into GitHub Actions
- [ ] Code coverage reports generated and published
- [ ] Code coverage threshold configured (80%)
- [ ] Environment approvals configured (Azure DevOps and GitHub)
- [ ] Quality gates understood (build validation, release gates)
- [ ] Testing pyramid understood with recommended test quantities

## Microsoft Learn References
- [Run tests in pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/javascript#run-unit-tests)
- [Review code coverage](https://learn.microsoft.com/en-us/azure/devops/pipelines/test/review-code-coverage-results)
- [Define approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals)
