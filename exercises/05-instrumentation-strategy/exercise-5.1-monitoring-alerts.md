# Exercise 5.1: Azure Monitor, Application Insights & Alerts

## Objective
Configure comprehensive monitoring for a DevOps environment using Azure Monitor, Application Insights, and related services.

## Skills Measured
- Configure Azure Monitor and Log Analytics to integrate with DevOps tools
- Configure collection of telemetry by using Application Insights, VM Insights, Container Insights, Storage Insights, and Network Insights
- Configure alerts for events in GitHub Actions and Azure Pipelines
- Inspect infrastructure performance indicators (CPU, memory, disk, network)

## Steps

### Part A: Set Up Application Insights

1. **Create Application Insights**:
   ```bash
   # Create Log Analytics workspace (required for App Insights)
   az monitor log-analytics workspace create \
     --resource-group rg-az400 \
     --workspace-name law-az400 \
     --location eastus

   # Create Application Insights
   az monitor app-insights component create \
     --app ai-az400-app \
     --location eastus \
     --resource-group rg-az400 \
     --workspace law-az400 \
     --kind web
   ```

2. **Instrument your Node.js app**:
   ```bash
   npm install applicationinsights
   ```

   Add to `hello.js` (at the very top):
   ```javascript
   const appInsights = require('applicationinsights');
   appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
     .setAutoCollectRequests(true)
     .setAutoCollectPerformance(true)
     .setAutoCollectExceptions(true)
     .setAutoCollectDependencies(true)
     .setAutoCollectConsole(true, true)
     .start();
   ```

3. **Configure App Service to use App Insights**:
   ```bash
   az webapp config appsettings set \
     --name az400app-dev \
     --resource-group rg-az400 \
     --settings APPLICATIONINSIGHTS_CONNECTION_STRING="InstrumentationKey=..."
   ```

### Part B: Azure Monitor Insights

1. **VM Insights** — monitor virtual machines:
   ```bash
   # Enable VM Insights
   az monitor vm-insights enable \
     --name agent-vm \
     --resource-group rg-az400 \
     --workspace law-az400
   ```
   Provides: CPU, memory, disk, network, process dependencies map

2. **Container Insights** — monitor AKS:
   ```bash
   # Enable Container Insights on AKS
   az aks enable-addons \
     --resource-group rg-az400 \
     --name aks-az400 \
     --addons monitoring \
     --workspace-resource-id /subscriptions/.../law-az400
   ```
   Provides: Container CPU/memory, pod health, node performance

3. **Storage Insights**: Monitor → Insights → Storage accounts
   - Shows: Availability, latency, capacity, transactions

4. **Network Insights**: Monitor → Insights → Networks
   - Shows: Network health, connectivity, traffic, security

### Part C: Create Alerts

1. **Metric alert** — high CPU:
   ```bash
   az monitor metrics alert create \
     --name "High CPU Alert" \
     --resource-group rg-az400 \
     --scopes /subscriptions/.../resourceGroups/rg-az400/providers/Microsoft.Web/sites/az400app-prod \
     --condition "avg CpuPercentage > 80" \
     --window-size 5m \
     --evaluation-frequency 1m \
     --action /subscriptions/.../actionGroups/az400-alerts
   ```

2. **Log alert** — application errors:
   ```bash
   # Create action group first
   az monitor action-group create \
     --name az400-alerts \
     --resource-group rg-az400 \
     --short-name az400 \
     --email-receivers devops-team=team@example.com
   ```

3. **Smart detection alerts** in Application Insights:
   - Automatically detects: performance anomalies, failure anomalies, memory leaks
   - Configure in: Application Insights → Smart Detection

4. **Pipeline failure alerts** in Azure DevOps:
   - Project Settings → Notifications → New subscription
   - Event: Build completed → Filter: Status = Failed
   - Deliver to: Team or custom email

5. **GitHub Actions failure notifications**:
   ```yaml
   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - name: Deploy
           run: ./deploy.sh

     notify:
       needs: deploy
       if: failure()
       runs-on: ubuntu-latest
       steps:
         - name: Notify on failure
           uses: slackapi/slack-github-action@v1
           with:
             payload: |
               {
                 "text": "Deployment failed! Workflow: ${{ github.workflow }}, Run: ${{ github.run_id }}"
               }
           env:
             SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
   ```

### Part D: Application Insights Features to Know

| Feature | Purpose |
|---------|---------|
| **Live Metrics** | Real-time telemetry stream |
| **Application Map** | Visualize component dependencies |
| **Failures** | Exceptions and failed requests |
| **Performance** | Response times and bottlenecks |
| **Availability** | URL ping tests (synthetic monitoring) |
| **Users/Sessions** | User analytics |
| **Distributed Tracing** | End-to-end transaction tracing |

### Part E: Create Availability Test

```bash
# Create a URL ping test
az monitor app-insights web-test create \
  --resource-group rg-az400 \
  --name "Health Check" \
  --defined-web-test-name "HealthPing" \
  --location "East US" \
  --tags "hidden-link:/subscriptions/.../ai-az400-app=Resource" \
  --web-test-kind "ping" \
  --synthetic-monitor-id "health-check-1" \
  --request-url "https://az400app-prod.azurewebsites.net/health" \
  --frequency 300 \
  --timeout 120 \
  --expected-status-code 200
```

## Validation Checklist

- [ ] Application Insights created and connected to Log Analytics
- [ ] Node.js app instrumented with App Insights SDK
- [ ] VM Insights, Container Insights concepts understood
- [ ] Metric alert created (CPU threshold)
- [ ] Log alert created (application errors)
- [ ] Pipeline failure notifications configured
- [ ] GitHub Actions failure notification workflow created
- [ ] Application Map and distributed tracing understood
- [ ] Availability test configured

## Microsoft Learn References
- [Application Insights overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Create metric alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-metric)
