# Exercise 1.5: Webhooks & Teams Integration

## Objective
Configure integrations between GitHub/Azure DevOps and Microsoft Teams using webhooks and connectors.

## Skills Measured
- Configure integration by using webhooks
- Configure integration between GitHub or Azure DevOps and Microsoft Teams

## Prerequisites
- GitHub repository from Exercise 1.1
- **Git** installed (`git --version` to verify)

## Steps

### Part A: GitHub Webhooks

1. **Create a webhook** in your GitHub repository:
   - Settings → Webhooks → Add webhook
   - Payload URL: Use https://webhook.site for testing (free temporary endpoint)
   - Content type: `application/json`
   - Secret: Generate a strong secret
   - Events: Select "Let me select individual events"
     - Push, Pull requests, Issues, Releases

2. **Trigger events** and inspect payloads:
   ```bash
   # Trigger a push event
   git commit --allow-empty -m "test: webhook trigger"
   git push

   # Create an issue to trigger issue event
   gh issue create --title "Test webhook" --body "Testing webhook delivery"
   ```

3. **Inspect the webhook delivery** at webhook.site and examine:
   - Headers (X-GitHub-Event, X-Hub-Signature-256)
   - Payload structure for each event type
   - Response timing

4. **Create a webhook receiver** (Azure Function):
   ```javascript
   // Azure Function — process GitHub webhook
   const crypto = require('crypto');

   module.exports = async function (context, req) {
       const secret = process.env.GITHUB_WEBHOOK_SECRET;
       const signature = req.headers['x-hub-signature-256'];
       const body = JSON.stringify(req.body);

       const hmac = crypto.createHmac('sha256', secret);
       const digest = 'sha256=' + hmac.update(body).digest('hex');

       if (signature !== digest) {
           context.res = { status: 401, body: 'Invalid signature' };
           return;
       }

       const event = req.headers['x-github-event'];
       context.log(`Received ${event} event`);

       // Process based on event type
       switch (event) {
           case 'push':
               context.log(`Push to ${req.body.ref} by ${req.body.pusher.name}`);
               break;
           case 'pull_request':
               context.log(`PR ${req.body.action}: ${req.body.pull_request.title}`);
               break;
           case 'issues':
               context.log(`Issue ${req.body.action}: ${req.body.issue.title}`);
               break;
       }

       context.res = { status: 200, body: 'Processed' };
   };
   ```

### Part B: Azure DevOps Service Hooks

1. **Configure service hooks** in Azure DevOps:
   - Project Settings → Service hooks → Create subscription
   - Available triggers: Build completed, Release deployment, Work item updated, PR created, Code pushed

2. **Create a webhook for build events**:
   - Service: Web Hooks
   - Trigger: Build completed
   - Filter: Status = Failed (alert on failures)
   - Action: Post via HTTP

3. **Create a webhook for work item events**:
   - Service: Web Hooks
   - Trigger: Work item updated
   - Filter: State changes to Closed

### Part C: Microsoft Teams Integration

1. **Install Azure DevOps app in Teams**:
   - In Teams → Apps → Search "Azure DevOps"
   - Install and sign in with your Azure DevOps account
   - Link to your project

2. **Configure notifications in a Teams channel**:
   ```
   @azure devops subscribe https://dev.azure.com/YOUR_ORG/AZ400-Prep
   ```
   Subscribe to events:
   - Build completed
   - Release deployment completed
   - Pull request created
   - Work item assigned

3. **Install GitHub app in Teams**:
   - In Teams → Apps → Search "GitHub"
   - Install and authenticate
   - In a channel:
   ```
   @github subscribe owner/repo issues pulls releases deployments
   ```

4. **Create an Incoming Webhook in Teams** (for custom notifications):
   - Channel → Connectors → Incoming Webhook
   - Name: "CI/CD Alerts"
   - Copy the webhook URL

5. **Send custom notifications from a pipeline**:
   ```yaml
   # Azure Pipelines task to send Teams notification
   - task: PowerShell@2
     displayName: 'Notify Teams'
     condition: always()
     inputs:
       targetType: 'inline'
       script: |
         $status = "$(Agent.JobStatus)"
         $color = if ($status -eq "Succeeded") { "00FF00" } else { "FF0000" }

         $body = @{
           "@type" = "MessageCard"
           "themeColor" = $color
           "title" = "Pipeline: $(Build.DefinitionName)"
           "text" = "Build #$(Build.BuildNumber) - $status"
           "sections" = @(
             @{
               "facts" = @(
                 @{ "name" = "Branch"; "value" = "$(Build.SourceBranchName)" }
                 @{ "name" = "Commit"; "value" = "$(Build.SourceVersion)" }
                 @{ "name" = "Author"; "value" = "$(Build.RequestedFor)" }
               )
             }
           )
           "potentialAction" = @(
             @{
               "@type" = "OpenUri"
               "name" = "View Build"
               "targets" = @(
                 @{ "os" = "default"; "uri" = "$(System.CollectionUri)$(System.TeamProject)/_build/results?buildId=$(Build.BuildId)" }
               )
             }
           )
         } | ConvertTo-Json -Depth 10

         Invoke-RestMethod -Uri "$(TeamsWebhookUrl)" -Method Post -ContentType "application/json" -Body $body
   ```

## Validation Checklist

- [ ] GitHub webhook configured and receiving events
- [ ] Webhook payload structure understood for push, PR, and issue events
- [ ] Webhook signature validation implemented
- [ ] Azure DevOps service hooks configured for build and work item events
- [ ] Azure DevOps Teams app installed and subscriptions configured
- [ ] GitHub Teams app installed and subscriptions configured
- [ ] Custom Teams notification sent from a pipeline

## Key Concepts to Review

- **Webhooks vs Service Hooks**: GitHub uses webhooks; Azure DevOps uses service hooks
- **Webhook security**: Signature validation with HMAC-SHA256
- **Teams message cards**: ActionCard format for rich notifications
- **Event-driven architecture**: How webhooks enable automation

## Microsoft Learn References
- [Service hooks in Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/service-hooks/overview)
- [Azure DevOps with Microsoft Teams](https://learn.microsoft.com/en-us/azure/devops/boards/integrations/boards-teams)
