# Exercise 3.9: Database Deployments & Self-Hosted Agents

## Objective
Implement database migrations in pipelines and configure self-hosted agents/runners.

## Skills Measured
- Implement a deployment that includes database tasks
- Design and implement a GitHub runner or Azure DevOps agent infrastructure

## Steps

### Part A: Database Migrations in Pipelines

1. **SQL Database migration with Azure Pipelines**:
   ```yaml
   stages:
     - stage: Deploy
       jobs:
         - deployment: DeployApp
           environment: 'Production'
           strategy:
             runOnce:
               deploy:
                 steps:
                   # Run database migrations BEFORE app deployment
                   - task: SqlAzureDacpacDeployment@1
                     inputs:
                       azureSubscription: 'Azure-Connection'
                       authenticationType: 'server'
                       serverName: 'sql-az400.database.windows.net'
                       databaseName: 'appdb'
                       sqlUsername: '$(DB_USERNAME)'
                       sqlPassword: '$(DB_PASSWORD)'
                       deployType: 'SqlTask'
                       sqlFile: 'database/migrations/V1__initial_schema.sql'
                     displayName: 'Run DB migration'

                   # Then deploy the application
                   - task: AzureWebApp@1
                     inputs:
                       azureSubscription: 'Azure-Connection'
                       appName: 'myapp-prod'
                       package: '$(Pipeline.Workspace)/drop/*.zip'
   ```

2. **Entity Framework migrations** (for .NET):
   ```yaml
   steps:
     - task: DotNetCoreCLI@2
       inputs:
         command: 'custom'
         custom: 'ef'
         arguments: 'database update --connection "$(ConnectionString)"'
       displayName: 'Run EF migrations'
   ```

3. **Flyway migrations** (platform-agnostic):
   ```yaml
   steps:
     - script: |
         flyway -url=jdbc:sqlserver://sql-az400.database.windows.net:1433;databaseName=appdb \
           -user=$(DB_USERNAME) \
           -password=$(DB_PASSWORD) \
           -locations=filesystem:database/migrations \
           migrate
       displayName: 'Run Flyway migrations'
   ```

4. **Database deployment best practices**:
   - Always run migrations before application deployment
   - Make migrations backward-compatible (expand-contract pattern)
   - Include rollback scripts
   - Test migrations against a copy of production data
   - Use transactions for atomicity

### Part B: Self-Hosted Azure DevOps Agents

1. **Set up a self-hosted agent on a VM**:
   ```bash
   # Create agent VM
   az vm create \
     --resource-group rg-az400 \
     --name agent-vm \
     --image Ubuntu2204 \
     --size Standard_D2s_v3 \
     --admin-username azureuser \
     --generate-ssh-keys

   # SSH into the VM
   ssh azureuser@<vm-ip>

   # Download and configure the agent
   mkdir myagent && cd myagent
   curl -O https://vstsagentpackage.azureedge.net/agent/3.232.1/vsts-agent-linux-x64-3.232.1.tar.gz
   tar zxvf vsts-agent-linux-x64-3.232.1.tar.gz
   ./config.sh
   # Enter: server URL, PAT token, agent pool name, agent name

   # Run as a service
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

2. **Use self-hosted agent in pipeline**:
   ```yaml
   pool:
     name: 'SelfHostedPool'  # Agent pool name
     demands:
       - npm
       - docker

   steps:
     - script: npm install
     - script: docker build -t myapp .
   ```

3. **Agent pool with Azure VM Scale Set** (auto-scaling):
   ```bash
   # Create VMSS for agents
   az vmss create \
     --resource-group rg-az400 \
     --name agent-vmss \
     --image Ubuntu2204 \
     --instance-count 0 \
     --upgrade-policy-mode manual
   ```
   Then configure in Azure DevOps: Project Settings → Agent Pools → New → Azure VM Scale Set

### Part C: GitHub Self-Hosted Runners

1. **Set up a self-hosted runner**:
   ```bash
   # Download runner
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
   tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

   # Configure
   ./config.sh --url https://github.com/YOUR_ORG --token YOUR_TOKEN

   # Install and start as service
   sudo ./svc.sh install
   sudo ./svc.sh start
   ```

2. **Use in workflow**:
   ```yaml
   jobs:
     build:
       runs-on: self-hosted
       # Or with labels:
       # runs-on: [self-hosted, linux, x64]
       steps:
         - uses: actions/checkout@v4
         - run: npm test
   ```

3. **Runner groups** (for GitHub Enterprise):
   - Organization → Settings → Actions → Runner groups
   - Restrict which repos can use which runners
   - Useful for security isolation

### Part D: Agent/Runner Decision Matrix

| Factor | Microsoft-Hosted | Self-Hosted |
|--------|-----------------|-------------|
| Cost | Pay per minute | VM/compute cost |
| Maintenance | None | You manage |
| Clean environment | Yes (fresh each run) | Persistent (cache benefit) |
| Network access | Public internet | Can access private networks |
| Custom software | Limited | Full control |
| Scaling | Automatic | Manual or VMSS |
| Best for | Standard builds | Private networks, special tools |

## Validation Checklist

- [ ] Database migration integrated into deployment pipeline
- [ ] Expand-contract migration pattern understood
- [ ] Self-hosted Azure DevOps agent configured
- [ ] Agent pool demands/capabilities understood
- [ ] GitHub self-hosted runner configured
- [ ] Microsoft-hosted vs self-hosted decision criteria memorized

## Microsoft Learn References
- [Self-hosted agents](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents)
- [GitHub self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners)
