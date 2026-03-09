# Exercise 3.6: Deployment Strategies

## Objective
Implement various deployment strategies including blue-green, canary, rolling, feature flags, and A/B testing.

## Skills Measured
- Design a deployment strategy (blue-green, canary, ring, progressive exposure, feature flags, A/B testing)
- Plan for minimizing downtime (VIP swap, load balancing, rolling deployments, deployment slots)
- Design a hotfix path plan
- Implement feature flags by using Azure App Configuration Feature Manager

## Prerequisites
- **Azure CLI (`az`)** — used to manage App Service deployment slots, traffic routing, and Azure App Configuration feature flags.
  - Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  - Authenticate: `az login`
- Azure subscription with an App Service (or you'll create one in the exercise)

## Steps

### Part A: Blue-Green Deployment (Azure App Service Slots)

```yaml
# cicd/deploy-blue-green.yaml
stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: npm ci && npm run build
          - publish: $(System.DefaultWorkingDirectory)
            artifact: app

  - stage: Deploy_BlueGreen
    dependsOn: Build
    jobs:
      - deployment: BlueGreen
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                # Deploy to staging slot (green)
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    appName: 'myapp-prod'
                    deployToSlotOrASE: true
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/app'

                # Warm up the staging slot
                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    action: 'Start Azure App Service'
                    webAppName: 'myapp-prod'
                    specifySlotOrASE: true
                    slotName: 'staging'

                # Run smoke tests against staging slot
                - script: |
                    response=$(curl -s -o /dev/null -w "%{http_code}" https://myapp-prod-staging.azurewebsites.net/health)
                    if [ "$response" != "200" ]; then
                      echo "Health check failed! Status: $response"
                      exit 1
                    fi
                  displayName: 'Smoke test staging slot'

                # Swap slots (instant zero-downtime switch)
                - task: AzureAppServiceManage@0
                  inputs:
                    azureSubscription: 'Azure-Connection'
                    action: 'Swap Slots'
                    webAppName: 'myapp-prod'
                    sourceSlot: 'staging'
                    targetSlot: 'production'
                  displayName: 'Swap staging ↔ production'
```

**How to set up blue-green in Azure:**
```bash
# Create an App Service with a deployment slot
az webapp deployment slot create \
  --name myapp-prod \
  --resource-group rg-az400 \
  --slot staging

# Configure slot-specific settings
az webapp config appsettings set \
  --name myapp-prod \
  --resource-group rg-az400 \
  --slot staging \
  --slot-settings ENVIRONMENT=staging
```

### Part B: Canary Deployment (Azure Traffic Routing)

```bash
# Route 10% of traffic to staging slot (canary)
az webapp traffic-routing set \
  --name myapp-prod \
  --resource-group rg-az400 \
  --distribution staging=10

# Monitor for errors...
# If OK, increase traffic
az webapp traffic-routing set \
  --name myapp-prod \
  --resource-group rg-az400 \
  --distribution staging=50

# If OK, swap to 100%
az webapp deployment slot swap \
  --name myapp-prod \
  --resource-group rg-az400 \
  --slot staging
```

Pipeline implementation:
```yaml
strategy:
  canary:
    increments: [10, 50]
    preDeploy:
      steps:
        - script: echo "Pre-deploy validation"
    deploy:
      steps:
        - script: echo "Deploy canary"
    routeTraffic:
      steps:
        - script: echo "Route $(strategy.increment)% traffic"
    postRouteTraffic:
      steps:
        - script: echo "Monitor canary for 5 minutes"
        - task: Delay@1
          inputs:
            delayForMinutes: '5'
    on:
      failure:
        steps:
          - script: echo "Canary failed — rolling back!"
      success:
        steps:
          - script: echo "Canary succeeded at $(strategy.increment)%"
```

### Part C: Rolling Deployment

```yaml
# Rolling deployment across multiple targets
strategy:
  rolling:
    maxParallel: 2  # Deploy to 2 VMs at a time
    preDeploy:
      steps:
        - script: echo "Remove from load balancer"
    deploy:
      steps:
        - script: echo "Deploy to $(Environment.ResourceName)"
    routeTraffic:
      steps:
        - script: echo "Re-add to load balancer"
    postRouteTraffic:
      steps:
        - script: echo "Health check $(Environment.ResourceName)"
    on:
      failure:
        steps:
          - script: echo "Rollback $(Environment.ResourceName)"
```

### Part D: Ring Deployment (Progressive Exposure)

```yaml
# Ring-based progressive deployment
stages:
  - stage: Ring0_Internal
    displayName: 'Ring 0 - Internal Users (1%)'
    jobs:
      - deployment: Ring0
        environment: 'ring-0-internal'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to internal ring"

  - stage: Ring1_Canary
    displayName: 'Ring 1 - Canary (5%)'
    dependsOn: Ring0_Internal
    jobs:
      - deployment: Ring1
        environment: 'ring-1-canary'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to 5% of users"

  - stage: Ring2_EarlyAdopters
    displayName: 'Ring 2 - Early Adopters (25%)'
    dependsOn: Ring1_Canary
    jobs:
      - deployment: Ring2
        environment: 'ring-2-early'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to 25% of users"

  - stage: Ring3_AllUsers
    displayName: 'Ring 3 - All Users (100%)'
    dependsOn: Ring2_EarlyAdopters
    jobs:
      - deployment: Ring3
        environment: 'Production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploy to all users"
```

### Part E: Feature Flags with Azure App Configuration

1. **Create Azure App Configuration**:
   ```bash
   az appconfig create \
     --name az400-appconfig \
     --resource-group rg-az400 \
     --location eastus

   # Create a feature flag
   az appconfig feature set \
     --name az400-appconfig \
     --feature BetaDashboard \
     --label Production

   # Enable/disable feature flag
   az appconfig feature enable --name az400-appconfig --feature BetaDashboard
   az appconfig feature disable --name az400-appconfig --feature BetaDashboard
   ```

2. **Use feature flags in Node.js**:
   ```javascript
   const { AppConfigurationClient } = require('@azure/app-configuration');
   const { FeatureManager, ConfigurationMapFeatureFlagProvider } = require('@microsoft/feature-management');

   const client = new AppConfigurationClient(process.env.APPCONFIG_CONNECTION_STRING);

   // Check feature flag
   async function isFeatureEnabled(featureName) {
     const featureFlag = await client.getConfigurationSetting({
       key: `.appconfig.featureflag/${featureName}`,
     });
     return JSON.parse(featureFlag.value).enabled;
   }

   // Usage in request handler
   app.get('/dashboard', async (req, res) => {
     if (await isFeatureEnabled('BetaDashboard')) {
       res.render('dashboard-v2');
     } else {
       res.render('dashboard-v1');
     }
   });
   ```

3. **Feature flag targeting** (percentage rollout, user groups):
   ```bash
   # Target specific users
   az appconfig feature filter add \
     --name az400-appconfig \
     --feature BetaDashboard \
     --filter-name Microsoft.Targeting \
     --filter-parameters Audience='{"Users":["user1@example.com"],"Groups":[{"Name":"BetaTesters","RolloutPercentage":100}],"DefaultRolloutPercentage":10}'
   ```

### Part F: Hotfix Path

```
Production bug detected!
        │
        ▼
   Create hotfix branch from main
        │
   git checkout -b hotfix/critical-fix main
        │
        ▼
   Fix the bug, add test
        │
        ▼
   Fast-track PR (streamlined review)
        │
        ▼
   Merge to main
        │
        ▼
   Auto-deploy through expedited pipeline
   (skip staging, deploy directly to prod with approval)
        │
        ▼
   Cherry-pick or merge back to develop/feature branches
```

## Validation Checklist

- [ ] Blue-green deployment with slot swap implemented
- [ ] Canary deployment with traffic routing configured
- [ ] Rolling deployment strategy understood
- [ ] Ring-based progressive exposure pipeline created
- [ ] Azure App Configuration feature flags created
- [ ] Feature flag used in application code
- [ ] Hotfix path documented and pipeline supports it
- [ ] Can explain when to use each deployment strategy

## Key Concept: Strategy Comparison

| Strategy | Downtime | Rollback Speed | Resource Cost | Complexity |
|----------|----------|----------------|---------------|------------|
| Blue-Green | Zero | Instant (swap) | 2x | Low |
| Canary | Zero | Fast | 1x + canary | Medium |
| Rolling | Minimal | Moderate | 1x | Medium |
| Ring | Zero | Fast | 1x+ | High |
| Feature Flags | Zero | Instant (toggle) | 1x | Low-Medium |

## Microsoft Learn References
- [Deployment strategies](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/deployment-jobs#deployment-strategies)
- [Azure App Configuration Feature Management](https://learn.microsoft.com/en-us/azure/azure-app-configuration/concept-feature-management)
- [Deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
