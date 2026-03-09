# Exercise 3.5: Pipeline Templates & Reusable Elements

## Objective
Create reusable pipeline elements including YAML templates, task groups, variable groups, and template expressions.

## Skills Measured
- Create reusable pipeline elements, including YAML templates, task groups, variables, and variable groups
- Develop and implement complex pipeline scenarios, such as hybrid pipelines, VM templates

## Prerequisites
- **Node.js & npm** installed (see Exercise 3.1)
- Azure DevOps project or GitHub repository with pipeline configuration from previous exercises

## Steps

### Part A: Step Templates

1. **Create a step template** (`cicd/templates/steps/build-node.yml`):
   ```yaml
   parameters:
     - name: nodeVersion
       type: string
       default: '20.x'
     - name: workingDirectory
       type: string
       default: '.'

   steps:
     - task: NodeTool@0
       inputs:
         versionSpec: ${{ parameters.nodeVersion }}
       displayName: 'Install Node.js ${{ parameters.nodeVersion }}'

     - script: npm ci
       workingDirectory: ${{ parameters.workingDirectory }}
       displayName: 'Install dependencies'

     - script: npm run build
       workingDirectory: ${{ parameters.workingDirectory }}
       displayName: 'Build application'
   ```

2. **Create a test step template** (`cicd/templates/steps/test-node.yml`):
   ```yaml
   parameters:
     - name: workingDirectory
       type: string
       default: '.'

   steps:
     - script: npm test
       workingDirectory: ${{ parameters.workingDirectory }}
       displayName: 'Run tests'

     - task: PublishTestResults@2
       inputs:
         testResultsFormat: 'JUnit'
         testResultsFiles: '**/junit.xml'
       condition: always()
       displayName: 'Publish test results'

     - task: PublishCodeCoverageResults@2
       inputs:
         summaryFileLocation: '${{ parameters.workingDirectory }}/coverage/cobertura-coverage.xml'
       condition: always()
       displayName: 'Publish code coverage'
   ```

### Part B: Job Templates

Create `cicd/templates/jobs/build-job.yml`:
```yaml
parameters:
  - name: pool
    type: object
    default:
      vmImage: 'ubuntu-latest'
  - name: nodeVersion
    type: string
    default: '20.x'
  - name: publishArtifact
    type: boolean
    default: true

jobs:
  - job: Build
    pool: ${{ parameters.pool }}
    steps:
      - template: ../steps/build-node.yml
        parameters:
          nodeVersion: ${{ parameters.nodeVersion }}

      - template: ../steps/test-node.yml

      - ${{ if parameters.publishArtifact }}:
        - task: ArchiveFiles@2
          inputs:
            rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
            archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'
          displayName: 'Archive application'

        - publish: $(Build.ArtifactStagingDirectory)
          artifact: drop
          displayName: 'Publish artifact'
```

### Part C: Stage Templates

Create `cicd/templates/stages/deploy-stage.yml`:
```yaml
parameters:
  - name: environment
    type: string
  - name: azureSubscription
    type: string
  - name: appName
    type: string
  - name: dependsOn
    type: object
    default: []
  - name: condition
    type: string
    default: 'succeeded()'
  - name: deployToSlot
    type: boolean
    default: false
  - name: slotName
    type: string
    default: 'staging'

stages:
  - stage: Deploy_${{ parameters.environment }}
    displayName: 'Deploy to ${{ parameters.environment }}'
    dependsOn: ${{ parameters.dependsOn }}
    condition: ${{ parameters.condition }}
    jobs:
      - deployment: Deploy
        environment: ${{ parameters.environment }}
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: ${{ parameters.azureSubscription }}
                    appType: 'webAppLinux'
                    appName: ${{ parameters.appName }}
                    package: '$(Pipeline.Workspace)/drop/*.zip'
                    ${{ if parameters.deployToSlot }}:
                      deployToSlotOrASE: true
                      slotName: ${{ parameters.slotName }}
```

### Part D: Main Pipeline Using Templates

Create `cicd/pipeline-with-templates.yaml`:
```yaml
trigger:
  branches:
    include:
      - main

variables:
  - template: templates/variables/common.yml
  - group: 'az400-variables'

stages:
  # Build stage using job template
  - stage: Build
    jobs:
      - template: templates/jobs/build-job.yml
        parameters:
          nodeVersion: '20.x'
          publishArtifact: true

  # Deploy stages using stage template
  - template: templates/stages/deploy-stage.yml
    parameters:
      environment: 'Development'
      azureSubscription: 'Azure-Connection'
      appName: 'myapp-dev'
      dependsOn: ['Build']

  - template: templates/stages/deploy-stage.yml
    parameters:
      environment: 'Staging'
      azureSubscription: 'Azure-Connection'
      appName: 'myapp-staging'
      dependsOn: ['Deploy_Development']

  - template: templates/stages/deploy-stage.yml
    parameters:
      environment: 'Production'
      azureSubscription: 'Azure-Connection'
      appName: 'myapp-prod'
      dependsOn: ['Deploy_Staging']
      deployToSlot: true
```

### Part E: Variable Templates

Create `cicd/templates/variables/common.yml`:
```yaml
variables:
  - name: vmImage
    value: 'ubuntu-latest'
  - name: nodeVersion
    value: '20.x'
  - name: azureRegion
    value: 'eastus'
  - name: appPrefix
    value: 'az400app'
```

### Part F: Conditional Insertion with Template Expressions

```yaml
parameters:
  - name: environments
    type: object
    default:
      - name: dev
        appName: myapp-dev
        approvalRequired: false
      - name: staging
        appName: myapp-staging
        approvalRequired: true
      - name: production
        appName: myapp-prod
        approvalRequired: true

stages:
  - stage: Build
    jobs:
      - job: Build
        steps:
          - script: echo "Building"

  - ${{ each env in parameters.environments }}:
    - stage: Deploy_${{ env.name }}
      displayName: 'Deploy to ${{ env.name }}'
      jobs:
        - deployment: Deploy
          environment: ${{ env.name }}
          strategy:
            runOnce:
              deploy:
                steps:
                  - script: echo "Deploying to ${{ env.appName }}"
```

### Part G: Cross-Repository Templates

```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: your-org/pipeline-templates
      ref: refs/heads/main
      endpoint: GitHub-Connection

stages:
  - template: stages/build.yml@templates
    parameters:
      nodeVersion: '20.x'
  - template: stages/deploy.yml@templates
    parameters:
      environment: 'production'
```

## Validation Checklist

- [ ] Step templates created and referenced
- [ ] Job templates created with parameters
- [ ] Stage templates created with conditional logic
- [ ] Variable templates created and shared
- [ ] `each` expression used to generate stages dynamically
- [ ] Conditional insertion (`${{ if }}`) used in templates
- [ ] Cross-repository template reference understood
- [ ] Main pipeline composes all templates together

## Microsoft Learn References
- [Template types & usage](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates)
- [Template expressions](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/template-expressions)
