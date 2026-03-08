# Exercise 4.1: Authentication & Authorization Methods

## Objective
Implement various authentication methods for Azure DevOps and GitHub, including Service Principals, Managed Identities, GitHub Apps, and PATs.

## Skills Measured
- Choose between Service Principals and Managed Identity (system-assigned and user-assigned)
- Implement and manage GitHub authentication (GitHub Apps, GITHUB_TOKEN, PATs)
- Implement and manage Azure DevOps service connections and PATs

## Steps

### Part A: Service Principals vs Managed Identity

1. **Create a Service Principal**:
   ```bash
   # Create SP with Contributor role
   az ad sp create-for-rbac \
     --name "az400-pipeline-sp" \
     --role Contributor \
     --scopes /subscriptions/<subscription-id>/resourceGroups/rg-az400 \
     --sdk-auth

   # Output contains: clientId, clientSecret, subscriptionId, tenantId
   # Store these as pipeline secrets!
   ```

2. **Create a Managed Identity**:
   ```bash
   # User-assigned managed identity
   az identity create \
     --name az400-pipeline-identity \
     --resource-group rg-az400

   # Assign role
   az role assignment create \
     --assignee <identity-principal-id> \
     --role Contributor \
     --scope /subscriptions/<sub-id>/resourceGroups/rg-az400
   ```

3. **Decision matrix** (memorize this!):

   | Criteria | Service Principal | Managed Identity |
   |----------|------------------|------------------|
   | Secret management | You manage client secret | No secret needed |
   | Rotation | Manual (secret expires) | Automatic |
   | Scope | Any Azure AD app | Azure resources only |
   | Use in pipelines | Yes (service connection) | Yes (with Azure-hosted agents) |
   | Use outside Azure | Yes | No |
   | Best for | CI/CD pipelines, multi-cloud | Azure-to-Azure, App Service |

   **System-assigned vs User-assigned Managed Identity**:
   | | System-assigned | User-assigned |
   |---|---|---|
   | Lifecycle | Tied to resource | Independent |
   | Sharing | One per resource | Shared across resources |
   | Use case | Single resource auth | Shared identity for multiple apps |

### Part B: Azure DevOps Service Connections

1. **Create a service connection** (Azure Resource Manager):
   - Project Settings → Service connections → New → Azure Resource Manager
   - Options:
     - **Service principal (automatic)**: Azure DevOps creates SP for you
     - **Service principal (manual)**: Use existing SP
     - **Managed identity**: Use VM's managed identity
     - **Workload identity federation**: Recommended — no secrets!

2. **Workload Identity Federation** (recommended):
   ```bash
   # This is the modern, secretless approach
   # Azure DevOps creates a federated credential on the SP
   # No client secret to rotate!
   ```

3. **Create a PAT** in Azure DevOps:
   - User Settings → Personal Access Tokens → New Token
   - Set scope (minimum permissions needed):
     - Code: Read
     - Build: Read & execute
     - Release: Read, write, & execute
   - Set expiration (max 1 year, recommend shorter)

### Part C: GitHub Authentication

1. **GITHUB_TOKEN** (automatic, per-workflow):
   ```yaml
   jobs:
     build:
       runs-on: ubuntu-latest
       permissions:
         contents: read       # Read repo
         packages: write      # Push packages
         pull-requests: write  # Comment on PRs
       steps:
         - uses: actions/checkout@v4
         - name: Use GITHUB_TOKEN
           run: |
             # GITHUB_TOKEN is automatically available
             gh pr comment $PR --body "Build passed!"
           env:
             GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
   ```

   **GITHUB_TOKEN permissions**:
   - Scoped to the repository
   - Expires when the job completes
   - Can be restricted with `permissions` key
   - Cannot trigger other workflows (prevents loops)

2. **Personal Access Tokens (PATs)**:
   - Settings → Developer settings → Personal access tokens
   - Fine-grained tokens (recommended) vs Classic tokens
   - Fine-grained: per-repository, specific permissions
   - Classic: broader scope, simpler but less secure

3. **GitHub Apps** (best for organization-level automation):
   ```yaml
   # Use a GitHub App for elevated permissions
   steps:
     - uses: actions/create-github-app-token@v1
       id: app-token
       with:
         app-id: ${{ vars.APP_ID }}
         private-key: ${{ secrets.APP_PRIVATE_KEY }}

     - uses: actions/checkout@v4
       with:
         token: ${{ steps.app-token.outputs.token }}
   ```

   **GitHub App vs PAT vs GITHUB_TOKEN**:
   | | GITHUB_TOKEN | PAT | GitHub App |
   |---|---|---|---|
   | Scope | Single repo | User-wide | Org/repo |
   | Expiration | Per job | Configurable | Long-lived (key) |
   | Can trigger workflows | No | Yes | Yes |
   | Rate limits | 1000/hr | 5000/hr | 15000/hr |
   | Best for | Standard CI | Cross-repo | Org automation |

### Part D: Azure DevOps Service Connections in Pipelines

```yaml
# Using Azure service connection
steps:
  - task: AzureCLI@2
    inputs:
      azureSubscription: 'My-Azure-Connection'  # Service connection name
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        az webapp list --resource-group rg-az400 --output table
```

## Validation Checklist

- [ ] Service Principal created with scoped permissions
- [ ] Managed Identity created and role-assigned
- [ ] SP vs Managed Identity decision matrix memorized
- [ ] Azure DevOps service connection created (workload identity federation preferred)
- [ ] PAT created with minimum required permissions
- [ ] GITHUB_TOKEN used with explicit permissions
- [ ] GitHub App vs PAT vs GITHUB_TOKEN differences understood
- [ ] System-assigned vs user-assigned managed identity differences known

## Microsoft Learn References
- [Service connections](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints)
- [Managed identities](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview)
- [GitHub Apps](https://docs.github.com/en/apps/creating-github-apps)
