# Exercise 1.2: Azure Boards & GitHub Integration

## Objective
Configure Azure Boards for work tracking and integrate with GitHub repositories for full traceability between code and work items.

## Skills Measured
- Design and implement integration for tracking work, including Azure Boards and repositories
- Design and implement source, bug, and quality traceability
- Configure integration between Azure Boards and GitHub repositories

## Prerequisites
- Azure DevOps organization (free at [dev.azure.com](https://dev.azure.com))
- GitHub repository from Exercise 1.1
- **Git** installed (`git --version` to verify) — used for commits with work item linking

## Steps

### Part A: Set Up Azure Boards

1. **Create an Azure DevOps project**:
   - Go to https://dev.azure.com
   - Create a new project: "AZ400-Prep"
   - Select **Agile** process template

2. **Configure the board**:
   - Navigate to Boards → Boards
   - Customize columns: New, Active, Resolved, Closed
   - Add swimlanes: Expedite, Standard
   - Configure WIP limits (e.g., In Progress = 3)

3. **Create work items**:
   - Create an **Epic**: "Application Modernization"
   - Create **Features** under the Epic:
     - "Containerize Application"
     - "Implement CI/CD Pipeline"
   - Create **User Stories** under each Feature:
     - "As a developer, I want a Dockerfile so I can build container images"
     - "As a developer, I want automated tests in the pipeline"
   - Create **Tasks** under each User Story

4. **Configure Sprints**:
   - Go to Project Settings → Team Configuration → Iterations
   - Create 3 sprints (2 weeks each)
   - Assign work items to sprints
   - Set capacity for team members

### Part B: Connect Azure Boards to GitHub

1. **Install the Azure Boards app on GitHub**:
   - Go to https://github.com/marketplace/azure-boards
   - Install it for your repository
   - Authorize and connect to your Azure DevOps organization

2. **Configure the connection**:
   - In Azure DevOps → Project Settings → GitHub connections
   - Verify the repository is connected
   - Choose which repositories to sync

3. **Link work items to GitHub commits and PRs**:
   ```bash
   # Reference Azure Boards work items in commits
   git commit -m "Add Dockerfile AB#123"

   # Use keywords to transition work items
   git commit -m "Fix input validation Fixes AB#125"
   ```

4. **Create a PR in GitHub** that references an Azure Boards work item:
   ```bash
   git checkout -b feature/containerize
   # Add a Dockerfile
   git add .
   git commit -m "Add Dockerfile for Node.js app AB#124"
   git push origin feature/containerize
   gh pr create --title "Containerize application" --body "Implements AB#124"
   ```

5. **Verify** in Azure Boards that:
   - The work item shows the linked GitHub commit
   - The work item shows the linked GitHub PR
   - The work item state transitions when the PR is merged

### Part C: Queries and Traceability

1. **Create queries in Azure Boards**:
   - "All open bugs assigned to me"
   - "Work items without linked PRs" (traceability gap query)
   - "Items completed this sprint"

   Example query (Shared Queries):
   ```
   Work Item Type = Bug
   AND State <> Closed
   AND Assigned To = @Me
   ```

2. **Create a traceability query**:
   ```
   Work Item Type IN (User Story, Bug)
   AND State = Active
   AND (Links.Count.Forward = 0 OR Links.Count.Reverse = 0)
   ```
   This finds work items without linked code changes — a traceability gap.

3. **Create a Dashboard widget**:
   - Go to Overview → Dashboards
   - Add widgets:
     - Sprint Burndown
     - Cumulative Flow Diagram
     - Query Results (your traceability query)
     - Velocity chart

### Part D: Configure Branch Policies with Work Item Linking

1. In Azure DevOps → Repos → Branches → main → Branch policies:
   - Enable **Check for linked work items** (Required)
   - This ensures every PR must reference a work item

## Validation Checklist

- [ ] Azure DevOps project created with Agile process
- [ ] Board customized with columns, swimlanes, and WIP limits
- [ ] Work item hierarchy created (Epic → Feature → User Story → Task)
- [ ] Sprints configured with capacity
- [ ] Azure Boards GitHub app installed and connected
- [ ] Commits and PRs successfully link to Azure Boards work items
- [ ] Work item state transitions on PR merge
- [ ] Traceability queries created
- [ ] Dashboard with burndown, CFD, and velocity widgets

## Key Concepts to Review

- **Process Templates**: Basic, Agile, Scrum, CMMI — know the differences
- **AB# syntax**: How to link GitHub commits/PRs to Azure Boards
- **Traceability**: Requirements → Code → Tests → Deployment
- **Cumulative Flow Diagram**: Shows bottlenecks and WIP

## Microsoft Learn References
- [Connect Azure Boards to GitHub](https://learn.microsoft.com/en-us/azure/devops/boards/github/)
- [About Azure Boards](https://learn.microsoft.com/en-us/azure/devops/boards/get-started/what-is-azure-boards)
