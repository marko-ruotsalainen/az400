# Exercise 3.8: Infrastructure as Code (IaC)

## Objective
Implement IaC using Bicep, ARM templates, and Terraform. Configure desired state with Azure Automation.

## Skills Measured
- Recommend a configuration management technology for application infrastructure
- Implement a configuration management strategy for application infrastructure
- Define an IaC strategy, including source control and automation of testing and deployment
- Design and implement desired state configuration (Azure Automation State Config, ARM, Bicep, Azure Automanage)

## Prerequisites
- **Azure CLI (`az`)** — used for Bicep deployments (`az deployment group create`) and what-if analysis.
  - Install: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
  - Authenticate: `az login`
  - Install Bicep: `az bicep install`
- **Terraform** — Infrastructure as Code tool by HashiCorp. Defines cloud resources in `.tf` files using HCL (HashiCorp Configuration Language).
  - Install: https://developer.hashicorp.com/terraform/install
  - Verify: `terraform --version`
- Azure subscription

## Steps

### Part A: Bicep Templates

1. **Create a Bicep file** (`infra/main.bicep`):
   ```bicep
   @description('The Azure region for resources')
   param location string = resourceGroup().location

   @description('The environment name')
   @allowed(['dev', 'staging', 'prod'])
   param environmentName string

   @description('The App Service Plan SKU')
   param skuName string = environmentName == 'prod' ? 'P1v3' : 'B1'

   var appName = 'az400app-${environmentName}'
   var appServicePlanName = 'asp-${appName}'
   var appInsightsName = 'ai-${appName}'

   resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
     name: appServicePlanName
     location: location
     sku: {
       name: skuName
     }
     kind: 'linux'
     properties: {
       reserved: true
     }
   }

   resource appService 'Microsoft.Web/sites@2023-01-01' = {
     name: appName
     location: location
     properties: {
       serverFarmId: appServicePlan.id
       siteConfig: {
         linuxFxVersion: 'NODE|20-lts'
         alwaysOn: environmentName == 'prod'
         appSettings: [
           {
             name: 'APPINSIGHTS_INSTRUMENTATIONKEY'
             value: appInsights.properties.InstrumentationKey
           }
           {
             name: 'ENVIRONMENT'
             value: environmentName
           }
         ]
       }
       httpsOnly: true
     }
   }

   // Add staging slot for production
   resource stagingSlot 'Microsoft.Web/sites/slots@2023-01-01' = if (environmentName == 'prod') {
     parent: appService
     name: 'staging'
     location: location
     properties: {
       serverFarmId: appServicePlan.id
     }
   }

   resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
     name: appInsightsName
     location: location
     kind: 'web'
     properties: {
       Application_Type: 'web'
     }
   }

   output appServiceUrl string = 'https://${appService.properties.defaultHostName}'
   output appInsightsKey string = appInsights.properties.InstrumentationKey
   ```

2. **Create a Bicep module** (`infra/modules/keyvault.bicep`):
   ```bicep
   param keyVaultName string
   param location string = resourceGroup().location
   param tenantId string = subscription().tenantId

   resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
     name: keyVaultName
     location: location
     properties: {
       tenantId: tenantId
       sku: {
         family: 'A'
         name: 'standard'
       }
       enableRbacAuthorization: true
       enableSoftDelete: true
       softDeleteRetentionInDays: 90
     }
   }

   output keyVaultUri string = keyVault.properties.vaultUri
   ```

3. **Deploy Bicep from pipeline**:
   ```yaml
   # Azure Pipelines
   steps:
     - task: AzureCLI@2
       inputs:
         azureSubscription: 'Azure-Connection'
         scriptType: 'bash'
         scriptLocation: 'inlineScript'
         inlineScript: |
           az deployment group create \
             --resource-group rg-az400-$(environment) \
             --template-file infra/main.bicep \
             --parameters environmentName=$(environment)
   ```

### Part B: Terraform

1. **Create Terraform config** (`infra/terraform/main.tf`):
   ```hcl
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~> 3.0"
       }
     }
     backend "azurerm" {
       resource_group_name  = "rg-terraform-state"
       storage_account_name = "tfstateaz400"
       container_name       = "tfstate"
       key                  = "terraform.tfstate"
     }
   }

   provider "azurerm" {
     features {}
   }

   variable "environment" {
     type    = string
     default = "dev"
   }

   variable "location" {
     type    = string
     default = "eastus"
   }

   resource "azurerm_resource_group" "main" {
     name     = "rg-az400-${var.environment}"
     location = var.location
   }

   resource "azurerm_service_plan" "main" {
     name                = "asp-az400-${var.environment}"
     location            = azurerm_resource_group.main.location
     resource_group_name = azurerm_resource_group.main.name
     os_type             = "Linux"
     sku_name            = var.environment == "prod" ? "P1v3" : "B1"
   }

   resource "azurerm_linux_web_app" "main" {
     name                = "az400app-${var.environment}"
     location            = azurerm_resource_group.main.location
     resource_group_name = azurerm_resource_group.main.name
     service_plan_id     = azurerm_service_plan.main.id

     site_config {
       application_stack {
         node_version = "20-lts"
       }
       always_on = var.environment == "prod"
     }

     app_settings = {
       "ENVIRONMENT" = var.environment
     }
   }

   output "app_url" {
     value = "https://${azurerm_linux_web_app.main.default_hostname}"
   }
   ```

2. **Terraform in pipeline**:
   ```yaml
   stages:
     - stage: Plan
       jobs:
         - job: TerraformPlan
           steps:
             - task: TerraformInstaller@0
               inputs:
                 terraformVersion: 'latest'
             - task: TerraformCLI@0
               inputs:
                 command: 'init'
                 workingDirectory: 'infra/terraform'
                 backendType: 'azurerm'
             - task: TerraformCLI@0
               inputs:
                 command: 'plan'
                 workingDirectory: 'infra/terraform'
                 commandOptions: '-out=tfplan'

     - stage: Apply
       dependsOn: Plan
       jobs:
         - deployment: TerraformApply
           environment: 'Production'
           strategy:
             runOnce:
               deploy:
                 steps:
                   - task: TerraformCLI@0
                     inputs:
                       command: 'apply'
                       workingDirectory: 'infra/terraform'
                       commandOptions: 'tfplan'
   ```

### Part C: IaC Testing

1. **Bicep linting and what-if**:
   ```bash
   # Lint Bicep
   az bicep lint --file infra/main.bicep

   # What-if (preview changes before deploying)
   az deployment group what-if \
     --resource-group rg-az400-dev \
     --template-file infra/main.bicep \
     --parameters environmentName=dev
   ```

2. **Terraform validation**:
   ```bash
   terraform fmt -check      # Check formatting
   terraform validate         # Validate syntax
   terraform plan             # Preview changes
   ```

### Part D: Configuration Management Comparison

| Tool | Type | Language | State | Best For |
|------|------|----------|-------|----------|
| **Bicep** | IaC | Bicep (DSL) | Azure-managed | Azure-only, simplicity |
| **ARM Templates** | IaC | JSON | Azure-managed | Azure-only, legacy |
| **Terraform** | IaC | HCL | Remote backend | Multi-cloud |
| **Azure Automation DSC** | Config Mgmt | PowerShell DSC | Pull server | VM configuration |
| **Azure Automanage** | Config Mgmt | Policies | Azure-managed | Best practice VM config |
| **Ansible** | Config Mgmt | YAML | Agentless | Linux servers |

### Part E: Desired State Configuration

```powershell
# Azure Automation State Configuration example
Configuration WebServerConfig {
    Node "localhost" {
        WindowsFeature IIS {
            Ensure = "Present"
            Name   = "Web-Server"
        }
        File WebContent {
            Ensure          = "Present"
            Type            = "Directory"
            DestinationPath = "C:\inetpub\wwwroot\app"
        }
    }
}
```

## Validation Checklist

- [ ] Bicep template created with parameters, variables, conditions, and modules
- [ ] Bicep deployed via Azure CLI
- [ ] Terraform config created with remote state backend
- [ ] Terraform plan and apply executed in pipeline
- [ ] Bicep what-if and Terraform plan used for change preview
- [ ] IaC testing (linting, validation) integrated into pipeline
- [ ] Configuration management tools compared and understood
- [ ] Desired State Configuration concept understood

## Microsoft Learn References
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Terraform on Azure](https://learn.microsoft.com/en-us/azure/developer/terraform/)
- [Azure Automation State Configuration](https://learn.microsoft.com/en-us/azure/automation/automation-dsc-overview)
