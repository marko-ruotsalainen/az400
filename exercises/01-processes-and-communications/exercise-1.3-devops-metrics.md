# Exercise 1.3: DevOps Metrics Dashboard

## Objective
Design and implement dashboards to track key DevOps metrics including DORA metrics, cycle time, lead time, and deployment frequency.

## Skills Measured
- Design and implement a dashboard, including flow of work, such as cycle times, time to recovery, and lead time
- Design and implement appropriate metrics and queries for project planning, development, testing, security, delivery, and operations

## Prerequisites
- Azure DevOps project from Exercise 1.2
- GitHub repository with some commit history
- **Azure CLI (`az`)** installed and authenticated — the official command-line tool for managing Azure resources. Used here to query pipeline run data.
  - Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  - Authenticate: `az login`
  - Verify: `az --version`

## Steps

### Part A: Understand DORA Metrics

Study the four key DORA (DevOps Research and Assessment) metrics:

| Metric | What It Measures | Elite | High | Medium | Low |
|--------|-----------------|-------|------|--------|-----|
| **Deployment Frequency** | How often you deploy | On-demand (multiple/day) | Weekly-monthly | Monthly-6 months | >6 months |
| **Lead Time for Changes** | Commit to production | <1 hour | 1 day-1 week | 1-6 months | >6 months |
| **Change Failure Rate** | % of deployments causing failure | 0-15% | 16-30% | 16-30% | >30% |
| **Time to Restore Service** | How long to recover | <1 hour | <1 day | 1 day-1 week | >6 months |

### Part B: Create Azure DevOps Analytics Dashboard

1. **Create a new dashboard** in Azure DevOps:
   - Overview → Dashboards → New Dashboard
   - Name: "DevOps Metrics"

2. **Add Cycle Time widget**:
   - Add widget → Analytics → Cycle Time
   - Configure for User Stories
   - Set time period: last 30 days
   - **Cycle Time** = Time from "Active" to "Closed"

3. **Add Lead Time widget**:
   - Add widget → Analytics → Lead Time
   - Configure for User Stories
   - **Lead Time** = Time from "New" (created) to "Closed"

4. **Add Velocity widget**:
   - Shows story points completed per sprint
   - Helps with sprint planning predictions

5. **Add Burndown chart**:
   - Sprint Burndown for current iteration
   - Shows remaining work vs. ideal trend

6. **Add Cumulative Flow Diagram**:
   - Shows work items in each state over time
   - Widening bands indicate bottlenecks

### Part C: Create Custom Queries for Metrics

1. **Deployment Frequency Query** (create via Azure DevOps REST API or pipeline):
   ```bash
   # Use Azure CLI to query pipeline runs
   az pipelines runs list --org https://dev.azure.com/YOUR_ORG --project AZ400-Prep --top 50 --output table

   # Count successful deployments per week
   az pipelines runs list --org https://dev.azure.com/YOUR_ORG --project AZ400-Prep \
     --result succeeded --query "[?finishedDate > '2026-02-01']" --output table
   ```

2. **Build Success Rate Query**:
   ```bash
   # Get build statistics
   az pipelines runs list --org https://dev.azure.com/YOUR_ORG --project AZ400-Prep \
     --query "[].{Name:definition.name, Result:result, Date:finishedDate}" --output table
   ```

3. **Create Analytics views** in Azure DevOps:
   - Boards → Analytics views → New view
   - Configure to include work items with history
   - This enables Power BI integration

### Part D: GitHub Insights

1. **Enable GitHub Insights** for your repository:
   - Go to repository → Insights tab
   - Review: Contributors, Commits, Code frequency, Dependency graph

2. **Use the Pulse view**:
   - Shows activity summary for a time period
   - PRs merged, issues closed, authors active

3. **Create GitHub Actions metrics** (add to your workflow):
   ```yaml
   # .github/workflows/metrics.yml
   name: Track Deployment Metrics
   on:
     workflow_run:
       workflows: ["Deploy"]
       types: [completed]

   jobs:
     track-metrics:
       runs-on: ubuntu-latest
       steps:
         - name: Record deployment result
           run: |
             echo "Workflow: ${{ github.event.workflow_run.name }}"
             echo "Result: ${{ github.event.workflow_run.conclusion }}"
             echo "Duration: ${{ github.event.workflow_run.run_started_at }} to ${{ github.event.workflow_run.updated_at }}"
             echo "Commit: ${{ github.event.workflow_run.head_sha }}"
   ```

### Part E: Build a Metrics Report

Create a summary document that would be relevant for a DevOps team review:

1. **Calculate manually** from your project data:
   - Average cycle time for completed stories
   - Average lead time for completed stories
   - Number of deployments this week
   - Build success rate (successful/total builds)

2. **Document** the metrics in a team-facing report format

## Validation Checklist

- [ ] Can explain all four DORA metrics and their significance
- [ ] Azure DevOps dashboard created with Cycle Time, Lead Time, Velocity, Burndown
- [ ] Cumulative Flow Diagram configured and understood
- [ ] Pipeline run queries executed via Azure CLI
- [ ] GitHub Insights reviewed (Pulse, Contributors, Code frequency)
- [ ] GitHub Actions workflow for tracking deployment metrics created
- [ ] Metrics report generated

## Key Concepts to Review

- **DORA Metrics**: The four key metrics and what elite performance looks like
- **Cycle Time vs Lead Time**: Cycle = active work; Lead = total from request to delivery
- **Cumulative Flow**: How to spot bottlenecks from band width changes
- **Value Stream Mapping**: Identifying waste in the delivery process

## Microsoft Learn References
- [Configure and manage Azure DevOps dashboards](https://learn.microsoft.com/en-us/azure/devops/report/dashboards/dashboards)
- [Cumulative flow and lead/cycle time](https://learn.microsoft.com/en-us/azure/devops/report/dashboards/cumulative-flow-cycle-lead-time-guidance)
