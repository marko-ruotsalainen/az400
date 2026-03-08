# Exercise 3.4: Azure Pipelines YAML — Multi-Stage Pipelines

## Objective
Build comprehensive multi-stage Azure Pipelines with YAML, including variables, templates, and complex scenarios.

## Skills Measured
- Design and implement integration between GitHub repositories and Azure Pipelines
- Develop pipelines by using YAML
- Design and implement a strategy for job execution order, including parallelism and multi-stage pipelines
- Develop and implement complex pipeline scenarios

## Steps

### Part A: Multi-Stage Pipeline

Create `cicd/multi-stage-pipeline.yaml`:
```yaml
trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - '*.md'
      - 'docs/**'

pr:
  branches:
    include:
      - main

variables:
  - name: vmImage
    value: 'ubuntu-latest'
  - name: nodeVersion
    value: '20.x'
  - group: 'az400-variables'  # Variable group reference

stages:
  # ==================== BUILD ====================
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: Build
        pool:
          vmImage: $(vmImage)
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(nodeVersion)

          - task: Cache@2
            inputs:
              key: 'npm | "$(Agent.OS)" | package-lock.json'
              path: '$(System.DefaultWorkingDirectory)/node_modules'
              cacheHitVar: 'CACHE_RESTORED'
            displayName: Cache npm packages

          - script: npm ci
            condition: ne(variables.CACHE_RESTORED, 'true')
            displayName: Install dependencies

          - script: npm test
            displayName: Run tests

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: '**/junit.xml'
            condition: always()

          - task: PublishCodeCoverageResults@2
            inputs:
              summaryFileLocation: '$(System.DefaultWorkingDirectory)/coverage/cobertura-coverage.xml'

          - script: |
              echo "##vso[build.updatebuildnumber]1.0.$(Build.BuildId)"
            displayName: Set build number

          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/app-$(Build.BuildNumber).zip'

          - publish: $(Build.ArtifactStagingDirectory)
            artifact: drop

  # ==================== DEV ====================
  - stage: Deploy_Dev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    variables:
      - name: environment
        value: 'dev'
    jobs:
      - deployment: DeployDev
        environment: 'Development'
        pool:
          vmImage: $(vmImage)
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    appType: 'webAppLinux'
                    appName: 'myapp-$(environment)'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
                    runtimeStack: 'NODE|20-lts'

  # ==================== STAGING ====================
  - stage: Deploy_Staging
    displayName: 'Deploy to Staging'
    dependsOn: Deploy_Dev
    jobs:
      - deployment: DeployStaging
        environment: 'Staging'  # Has approval check
        pool:
          vmImage: $(vmImage)
        strategy:
          runOnce:
            preDeploy:
              steps:
                - script: echo "Pre-deployment validation"
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    appType: 'webAppLinux'
                    appName: 'myapp-staging'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
            routeTraffic:
              steps:
                - script: echo "Routing 10% traffic to new deployment"
            postRouteTraffic:
              steps:
                - script: echo "Running smoke tests"
            on:
              failure:
                steps:
                  - script: echo "Rolling back..."
              success:
                steps:
                  - script: echo "Staging deployment succeeded"

  # ==================== PRODUCTION ====================
  - stage: Deploy_Production
    displayName: 'Deploy to Production'
    dependsOn: Deploy_Staging
    jobs:
      - deployment: DeployProd
        environment: 'Production'  # Has approval + business hours check
        pool:
          vmImage: $(vmImage)
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    appType: 'webAppLinux'
                    appName: 'myapp-prod'
                    package: '$(Pipeline.Workspace)/drop/*.zip'
                    deployToSlotOrASE: true
                    slotName: 'staging'
                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    action: 'Swap Slots'
                    webAppName: 'myapp-prod'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
```

### Part B: Variables and Variable Groups

1. **Variable types in Azure Pipelines**:
   ```yaml
   variables:
     # Inline variable
     - name: myVar
       value: 'hello'

     # Variable group (defined in Library)
     - group: 'my-variable-group'

     # Template variable file
     - template: vars/common.yml

     # Conditional variable
     - ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
       - name: environment
         value: 'production'
     - ${{ else }}:
       - name: environment
         value: 'development'
   ```

2. **Create a variable group** in Azure DevOps:
   - Pipelines → Library → Variable groups → New
   - Name: "az400-variables"
   - Add variables:
     - `APP_NAME` = myapp
     - `AZURE_REGION` = eastus
   - Link to Azure Key Vault for secrets

3. **Runtime expressions vs compile-time**:
   ```yaml
   # Compile-time expression (evaluated before pipeline runs)
   ${{ variables.myVar }}

   # Runtime expression (evaluated during pipeline execution)
   $(myVar)

   # Runtime expression in conditions
   $[variables.myVar]
   ```

### Part C: Pipeline Parameters

```yaml
parameters:
  - name: environment
    displayName: 'Target Environment'
    type: string
    default: 'dev'
    values: ['dev', 'staging', 'production']

  - name: runTests
    displayName: 'Run Tests'
    type: boolean
    default: true

  - name: regions
    displayName: 'Deployment Regions'
    type: object
    default:
      - eastus
      - westeurope

stages:
  - stage: Test
    condition: eq('${{ parameters.runTests }}', true)
    jobs:
      - job: Test
        steps:
          - script: npm test

  - ${{ each region in parameters.regions }}:
    - stage: Deploy_${{ region }}
      displayName: 'Deploy to ${{ region }}'
      jobs:
        - job: Deploy
          steps:
            - script: echo "Deploying to ${{ region }}"
```

### Part D: Connect GitHub Repo to Azure Pipelines

1. **Create a service connection to GitHub**:
   - Project Settings → Service connections → New → GitHub
   - Authenticate with GitHub
   - Name: "GitHub-Connection"

2. **Create pipeline from GitHub repo**:
   ```bash
   # Or use Azure DevOps UI:
   # Pipelines → New Pipeline → GitHub → Select repo → Existing YAML file
   az pipelines create \
     --name "CI-CD" \
     --repository your-org/your-repo \
     --repository-type github \
     --branch main \
     --yml-path cicd/multi-stage-pipeline.yaml
   ```

## Validation Checklist

- [ ] Multi-stage pipeline created (Build → Dev → Staging → Production)
- [ ] Deployment jobs with environments configured
- [ ] Pipeline caching implemented
- [ ] Variable groups created and referenced
- [ ] Compile-time vs runtime expressions understood
- [ ] Pipeline parameters implemented with conditions
- [ ] `each` expression used for dynamic stage generation
- [ ] GitHub repository connected to Azure Pipelines
- [ ] Deployment strategy lifecycle hooks used (preDeploy, deploy, routeTraffic, etc.)

## Microsoft Learn References
- [YAML pipeline schema](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/)
- [Multi-stage pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/stages)
- [Variables](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/variables)
