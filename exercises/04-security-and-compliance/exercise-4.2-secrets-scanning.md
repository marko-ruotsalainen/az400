# Exercise 4.2: Secrets Management & Security Scanning

## Objective
Manage secrets securely in pipelines and implement automated security scanning with GitHub Advanced Security.

## Skills Measured
- Implement and manage secrets, keys, and certificates by using Azure Key Vault
- Implement and manage secrets in GitHub Actions and Azure Pipelines
- Design and implement a strategy for managing sensitive files during deployment
- Design pipelines to prevent leakage of sensitive information
- Design a strategy for security and compliance scanning
- Configure GitHub Advanced Security for both GitHub and Azure DevOps
- Automate container scanning and analysis of open-source components

## Prerequisites
- **Azure CLI (`az`)** — used to create and manage Azure Key Vault and secrets.
  - Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  - Authenticate: `az login`
- **GitHub CLI (`gh`)** — used to manage GitHub repository secrets.
  - Install: https://cli.github.com/
  - Authenticate: `gh auth login`
- **Node.js & npm/npx** — used for security scanning tools like `npm audit` and `license-checker`.
  - Install: https://nodejs.org/ (LTS version)
- **Docker** (optional) — for container image scanning.
  - Install: https://docs.docker.com/get-docker/

## Steps

### Part A: Azure Key Vault Integration

1. **Create Key Vault and add secrets**:
   ```bash
   az keyvault create \
     --name kv-az400 \
     --resource-group rg-az400 \
     --location eastus

   az keyvault secret set --vault-name kv-az400 --name "DbConnectionString" --value "Server=..."
   az keyvault secret set --vault-name kv-az400 --name "ApiKey" --value "sk-..."
   ```

2. **Azure Pipelines — Key Vault variable group**:
   - Pipelines → Library → Variable groups → New
   - Toggle "Link secrets from an Azure Key Vault"
   - Select subscription, Key Vault
   - Authorize and add secrets

   ```yaml
   variables:
     - group: 'keyvault-secrets'  # Linked to Key Vault

   steps:
     - script: |
         # Secrets are available as variables
         echo "Deploying with connection: $(DbConnectionString)"
       displayName: 'Use Key Vault secrets'
       # Azure Pipelines automatically masks secret values in logs
   ```

3. **Azure Pipelines — AzureKeyVault task**:
   ```yaml
   steps:
     - task: AzureKeyVault@2
       inputs:
         azureSubscription: 'Azure-Connection'
         keyVaultName: 'kv-az400'
         secretsFilter: 'DbConnectionString,ApiKey'
         runAsPreJob: true
       displayName: 'Fetch secrets from Key Vault'

     - script: echo "Using secret (masked in logs)"
       env:
         DB_CONN: $(DbConnectionString)
   ```

4. **GitHub Actions — Key Vault secrets**:
   ```yaml
   steps:
     - uses: azure/login@v2
       with:
         creds: ${{ secrets.AZURE_CREDENTIALS }}

     - uses: azure/get-keyvault-secrets@v1
       with:
         keyvault: kv-az400
         secrets: 'DbConnectionString,ApiKey'
       id: kv-secrets

     - run: echo "Secret fetched (masked)"
       env:
         DB_CONN: ${{ steps.kv-secrets.outputs.DbConnectionString }}
   ```

### Part B: Pipeline Secrets Best Practices

1. **Prevent secret leakage**:
   ```yaml
   # Azure Pipelines — secrets are auto-masked in logs
   # But be careful with:
   steps:
     - script: |
         # BAD: Writing to file that could be published
         # echo $(secret) > config.txt

         # GOOD: Use environment variables
         export DB_CONN="$(DbConnectionString)"
         node deploy.js
   ```

2. **Secure files** in Azure Pipelines:
   - Pipelines → Library → Secure files
   - Upload: certificates, SSH keys, signing keys
   ```yaml
   steps:
     - task: DownloadSecureFile@1
       name: sshKey
       inputs:
         secureFile: 'id_rsa'
     - script: |
         mkdir -p ~/.ssh
         cp $(sshKey.secureFilePath) ~/.ssh/id_rsa
         chmod 600 ~/.ssh/id_rsa
   ```

3. **GitHub Actions — encrypted secrets**:
   ```bash
   # Set secrets via CLI
   gh secret set DB_PASSWORD --body "secret-value"
   gh secret set --env production DB_PASSWORD --body "prod-secret"
   ```

### Part C: GitHub Advanced Security (GHAS)

1. **Enable GitHub Advanced Security**:
   - Repository → Settings → Security → Code security and analysis
   - Enable:
     - Dependabot alerts
     - Dependabot security updates
     - Code scanning (CodeQL)
     - Secret scanning

2. **Dependabot configuration** (`.github/dependabot.yml`):
   ```yaml
   version: 2
   updates:
     - package-ecosystem: "npm"
       directory: "/"
       schedule:
         interval: "weekly"
         day: "monday"
       open-pull-requests-limit: 10
       labels:
         - "dependencies"
       commit-message:
         prefix: "chore(deps)"
       groups:
         dev-dependencies:
           dependency-type: "development"
         production-dependencies:
           dependency-type: "production"

     - package-ecosystem: "github-actions"
       directory: "/"
       schedule:
         interval: "weekly"
   ```

3. **CodeQL code scanning** (`.github/workflows/codeql.yml`):
   ```yaml
   name: CodeQL Analysis
   on:
     push:
       branches: [main]
     pull_request:
       branches: [main]
     schedule:
       - cron: '0 6 * * 1'  # Weekly Monday 6 AM

   jobs:
     analyze:
       runs-on: ubuntu-latest
       permissions:
         actions: read
         contents: read
         security-events: write
       strategy:
         matrix:
           language: [javascript]
       steps:
         - uses: actions/checkout@v4

         - name: Initialize CodeQL
           uses: github/codeql-action/init@v3
           with:
             languages: ${{ matrix.language }}
             queries: security-extended

         - name: Autobuild
           uses: github/codeql-action/autobuild@v3

         - name: Perform CodeQL Analysis
           uses: github/codeql-action/analyze@v3
           with:
             category: "/language:${{ matrix.language }}"
   ```

4. **Secret scanning** — automatically detects:
   - API keys, tokens, connection strings
   - Push protection: blocks commits containing secrets
   - Enable push protection: Settings → Security → Secret scanning → Push protection

5. **Container scanning** in GitHub Actions:
   ```yaml
   steps:
     - name: Build container image
       run: docker build -t myapp:latest .

     - name: Run Trivy vulnerability scanner
       uses: aquasecurity/trivy-action@master
       with:
         image-ref: 'myapp:latest'
         format: 'sarif'
         output: 'trivy-results.sarif'
         severity: 'CRITICAL,HIGH'

     - name: Upload scan results
       uses: github/codeql-action/upload-sarif@v3
       with:
         sarif_file: 'trivy-results.sarif'
   ```

### Part D: GitHub Advanced Security for Azure DevOps (GHAzDO)

1. **Enable in Azure DevOps**:
   - Organization Settings → General → GitHub Advanced Security for Azure DevOps
   - Enable per-repository or organization-wide

2. **Configure in pipeline**:
   ```yaml
   steps:
     - task: AdvancedSecurity-Dependency-Scanning@1
       displayName: 'Dependency scanning'

     - task: AdvancedSecurity-CodeQL-Init@1
       inputs:
         languages: 'javascript'
       displayName: 'CodeQL Initialize'

     - task: AdvancedSecurity-CodeQL-Autobuild@1
       displayName: 'CodeQL Autobuild'

     - task: AdvancedSecurity-CodeQL-Analyze@1
       displayName: 'CodeQL Analyze'

     - task: AdvancedSecurity-Publish@1
       displayName: 'Publish security results'
   ```

### Part E: Microsoft Defender for Cloud DevOps Security

1. **Connect to Defender for Cloud**:
   - Defender for Cloud → Environment settings → Add environment → Azure DevOps / GitHub
   - Authorize Defender for Cloud to access your repos
   - Enable DevOps Security posture

2. **Key capabilities**:
   - Unified visibility across GitHub and Azure DevOps
   - Infrastructure-as-code scanning
   - Secret detection
   - Dependency vulnerability scanning
   - Security posture recommendations

## Validation Checklist

- [ ] Key Vault created with secrets
- [ ] Key Vault linked to Azure Pipelines variable group
- [ ] AzureKeyVault task used in pipeline
- [ ] Key Vault secrets accessed in GitHub Actions
- [ ] Secure files used for certificates/keys
- [ ] Pipeline configured to prevent secret leakage
- [ ] Dependabot configured for multiple ecosystems
- [ ] CodeQL code scanning enabled and running
- [ ] Secret scanning with push protection enabled
- [ ] Container scanning implemented
- [ ] GHAzDO tasks configured in Azure Pipelines
- [ ] Defender for Cloud DevOps integration understood

## Microsoft Learn References
- [Azure Key Vault in pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault)
- [GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)
- [GHAzDO](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features)
- [Defender for Cloud DevOps](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction)
