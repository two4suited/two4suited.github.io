---
title: "Building a Modern Development Platform: Terraform & Terraform Cloud for Azure Infrastructure 🏗️"
date: 2025-10-21T06:00:00-07:00
draft: false
categories: ["platform","terraform","iac","azure","infrastructure","terraform-cloud"]
description: "Setting up Terraform Cloud with Azure service principals, building reusable modules based on Azure's resource organization patterns, and publishing to a private registry"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction 🚀

In our [tool selection post](https://brianpsheridan.com/platform/tools/modernization/2025/10/04/tool-selection.html), we chose Terraform as our Infrastructure as Code (IaC) tool. But choosing the tool is just the beginning. To build a truly modern development platform, we need:

- **Secure Azure authentication** without storing credentials in code
- **Centralized state management** that works across teams
- **Reusable infrastructure patterns** based on Azure best practices
- **A module registry** for sharing standardized components

This post walks through setting up **Terraform Cloud** with Azure, organizing infrastructure code, and building modules that align with Azure's resource classification patterns from [Azure Charts](https://azurecharts.com/).

## Why Terraform Cloud? ☁️

Before diving into the setup, let's understand why we're using Terraform Cloud instead of local state files or basic remote backends:

**State Management**
- Centralized, secure state storage with automatic locking
- State versioning and rollback capabilities
- Team collaboration without state file conflicts

**Secure Credential Management**
- Workload Identity Federation with Azure (no stored secrets!)
- Encrypted variable storage
- Audit logging for compliance

**Team Workflows**
- VCS-driven runs with GitHub integration
- Policy enforcement with Sentinel
- Private module registry for code reuse

**Cost Optimization**
- Free tier supports up to 500 resources per month
- Pay only for what you use beyond that

## Terraform Cloud Setup 🛠️

### Creating Your Organization

First, sign up at [Terraform Cloud](https://app.terraform.io/) and create an organization. This will be the container for all your projects and workspaces.

```bash
# After signing up, note your organization name
# We'll use "my-org" as an example
export TF_CLOUD_ORG="my-org"
```

### Understanding Projects and Workspaces

Terraform Cloud uses a two-level hierarchy:

**Projects**: Logical groupings of related workspaces (e.g., "Platform Infrastructure", "Application Environments")

**Workspaces**: Individual environment configurations (e.g., "shared-services-dev", "app-prod", "app-staging")

This maps well to our platform structure:

```
Organization: my-org
├── Project: Platform Infrastructure
│   ├── Workspace: shared-services-dev
│   ├── Workspace: shared-services-staging
│   └── Workspace: shared-services-prod
└── Project: Application Environments
    ├── Workspace: app-dev
    ├── Workspace: app-staging
    └── Workspace: app-prod
```

### Creating Projects

In Terraform Cloud UI:

1. Navigate to **Projects** → **New Project**
2. Create "Platform Infrastructure" project
3. Create "Application Environments" project

Or use the Terraform CLI:

```bash
# Login to Terraform Cloud
terraform login

# Projects are created automatically when you create workspaces
# We'll create workspaces in the next section
```

## Azure Service Principal Setup 🔐

Now comes the critical part: setting up secure authentication between Terraform Cloud and Azure. We'll use **Workload Identity Federation** instead of storing client secrets.

### Step 1: Create the Service Principal

```bash
# Set your Azure subscription
SUBSCRIPTION_ID="your-subscription-id"
az account set --subscription $SUBSCRIPTION_ID

# Create a service principal for Terraform
SP_NAME="terraform-platform-sp"

# Create with no role assignment initially
az ad sp create-for-rbac \
  --name $SP_NAME \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json > sp-output.json

# Capture the important values
CLIENT_ID=$(cat sp-output.json | jq -r '.appId')
TENANT_ID=$(cat sp-output.json | jq -r '.tenant')

echo "Client ID: $CLIENT_ID"
echo "Tenant ID: $TENANT_ID"

# Clean up the file (it contains a secret we won't use)
rm sp-output.json
```

### Step 2: Configure Workload Identity Federation

This is where the magic happens - no stored secrets!

```bash
# Get your Terraform Cloud organization name
TF_CLOUD_ORG="my-org"

# Create federated credential for the service principal
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "terraform-cloud-federation",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:*:workspace:*:run_phase:*",
    "description": "Terraform Cloud workload identity",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**What's happening here?**

- Terraform Cloud gets a token from Azure AD when runs execute
- Azure validates the token matches the subject pattern (org/project/workspace)
- No client secret needed - the trust is based on the issuer (Terraform Cloud)

### Step 3: Assign Azure Permissions

Grant the service principal appropriate permissions:

```bash
# Contributor at subscription level
az role assignment create \
  --assignee $CLIENT_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"

# Optional: Add User Access Administrator if managing RBAC
az role assignment create \
  --assignee $CLIENT_ID \
  --role "User Access Administrator" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### Step 4: Configure Terraform Cloud Workspace Variables

For each workspace, configure these environment variables:

In Terraform Cloud UI → Workspace → Variables:

```hcl
# Azure Authentication (Environment Variables)
ARM_CLIENT_ID           = "<client-id>"        # From step 1
ARM_SUBSCRIPTION_ID     = "<subscription-id>"
ARM_TENANT_ID          = "<tenant-id>"         # From step 1
ARM_USE_OIDC           = "true"               # Enable OIDC
```

**Important**: Mark these as environment variables, NOT Terraform variables. Also mark sensitive variables as sensitive.

No `ARM_CLIENT_SECRET` needed! 🎉

## Infrastructure Code Organization 📁

Now let's structure our Terraform code. We'll create an `infra` folder with two main areas:

```
infra/
├── shared/                      # Shared services infrastructure
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf            # Backend config
│   └── modules/
│       ├── identity-module/     # From azurecharts.com
│       ├── integration-module/
│       ├── networking-module/
│       └── storage-module/
└── app/                         # Application infrastructure
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── terraform.tf
    └── modules/
        ├── compute-module/      # From azurecharts.com
        ├── container-module/
        └── database-module/
```

### Shared Infrastructure (`infra/shared`)

This contains resources shared across all applications:

**`infra/shared/terraform.tf`**
```hcl
terraform {
  required_version = ">= 1.6"

  cloud {
    organization = "my-org"
    
    workspaces {
      name = "shared-services-dev"
    }
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
  
  # OIDC authentication - no client secret needed!
  use_oidc = true
}
```

**`infra/shared/main.tf`**
```hcl
# Use modules for standardized resource groups
module "networking" {
  source = "./modules/networking-module"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
}

module "identity" {
  source = "./modules/identity-module"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
}

module "integration" {
  source = "./modules/integration-module"
  
  environment        = var.environment
  location          = var.location
  tags              = var.tags
  vnet_id           = module.networking.vnet_id
  subnet_ids        = module.networking.subnet_ids
}

module "storage" {
  source = "./modules/storage-module"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
}
```

### Application Infrastructure (`infra/app`)

This contains application-specific resources:

**`infra/app/terraform.tf`**
```hcl
terraform {
  required_version = ">= 1.6"

  cloud {
    organization = "my-org"
    
    workspaces {
      name = "app-dev"
    }
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
  use_oidc = true
}
```

**`infra/app/main.tf`**
```hcl
# Reference shared infrastructure outputs
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-services-${var.environment}"
    }
  }
}

module "compute" {
  source = "./modules/compute-module"
  
  environment     = var.environment
  location        = var.location
  tags            = var.tags
  subnet_id       = data.terraform_remote_state.shared.outputs.app_subnet_id
  acr_id          = data.terraform_remote_state.shared.outputs.acr_id
}

module "containers" {
  source = "./modules/container-module"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
  subnet_id   = data.terraform_remote_state.shared.outputs.container_subnet_id
}

module "database" {
  source = "./modules/database-module"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
  subnet_id   = data.terraform_remote_state.shared.outputs.db_subnet_id
}
```

## Azure Charts Resource Classification 📊

[Azure Charts](https://azurecharts.com/) provides an excellent resource classification system. We've modeled our modules after their groupings:

### Identity Module (Identity & Access Management)
```hcl
# infra/shared/modules/identity-module/main.tf
resource "azurerm_resource_group" "identity" {
  name     = "rg-${var.environment}-identity"
  location = var.location
  tags     = var.tags
}

# Key Vault for secrets
resource "azurerm_key_vault" "main" {
  name                = "kv-${var.environment}-${random_string.suffix.result}"
  location            = azurerm_resource_group.identity.location
  resource_group_name = azurerm_resource_group.identity.name
  tenant_id           = data.azurerm_client_config.current.tenant_id
  sku_name           = "standard"
  
  # Enable for Azure RBAC
  enable_rbac_authorization = true
  
  tags = var.tags
}

# Managed Identities
resource "azurerm_user_assigned_identity" "app_identity" {
  name                = "id-${var.environment}-app"
  resource_group_name = azurerm_resource_group.identity.name
  location            = azurerm_resource_group.identity.location
  tags                = var.tags
}
```

### Networking Module
```hcl
# infra/shared/modules/networking-module/main.tf
resource "azurerm_resource_group" "networking" {
  name     = "rg-${var.environment}-networking"
  location = var.location
  tags     = var.tags
}

# Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  tags                = var.tags
}

# Subnets for different tiers
resource "azurerm_subnet" "app" {
  name                 = "snet-${var.environment}-app"
  resource_group_name  = azurerm_resource_group.networking.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "container" {
  name                 = "snet-${var.environment}-container"
  resource_group_name  = azurerm_resource_group.networking.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  
  delegation {
    name = "container-delegation"
    service_delegation {
      name = "Microsoft.ContainerInstance/containerGroups"
    }
  }
}

resource "azurerm_subnet" "database" {
  name                 = "snet-${var.environment}-db"
  resource_group_name  = azurerm_resource_group.networking.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.3.0/24"]
}

# Network Security Group
resource "azurerm_network_security_group" "app" {
  name                = "nsg-${var.environment}-app"
  location            = azurerm_resource_group.networking.location
  resource_group_name = azurerm_resource_group.networking.name
  tags                = var.tags
}
```

### Integration Module (Event Grid, Service Bus, API Management)
```hcl
# infra/shared/modules/integration-module/main.tf
resource "azurerm_resource_group" "integration" {
  name     = "rg-${var.environment}-integration"
  location = var.location
  tags     = var.tags
}

# Service Bus Namespace
resource "azurerm_servicebus_namespace" "main" {
  name                = "sb-${var.environment}-${random_string.suffix.result}"
  location            = azurerm_resource_group.integration.location
  resource_group_name = azurerm_resource_group.integration.name
  sku                 = "Standard"
  tags                = var.tags
}

# API Management (optional, for dev use Developer tier)
resource "azurerm_api_management" "main" {
  count               = var.enable_apim ? 1 : 0
  name                = "apim-${var.environment}-${random_string.suffix.result}"
  location            = azurerm_resource_group.integration.location
  resource_group_name = azurerm_resource_group.integration.name
  publisher_name      = var.publisher_name
  publisher_email     = var.publisher_email
  sku_name            = var.environment == "prod" ? "Standard_1" : "Developer_1"
  tags                = var.tags
}
```

### Storage Module
```hcl
# infra/shared/modules/storage-module/main.tf
resource "azurerm_resource_group" "storage" {
  name     = "rg-${var.environment}-storage"
  location = var.location
  tags     = var.tags
}

# Azure Container Registry
resource "azurerm_container_registry" "main" {
  name                = "acr${var.environment}${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.storage.name
  location            = azurerm_resource_group.storage.location
  sku                 = "Standard"
  admin_enabled       = false
  tags                = var.tags
}

# Storage Account
resource "azurerm_storage_account" "main" {
  name                     = "st${var.environment}${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.storage.name
  location                 = azurerm_resource_group.storage.location
  account_tier             = "Standard"
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  tags                     = var.tags
}
```

### Compute Module (App Service, Functions, VMs)
```hcl
# infra/app/modules/compute-module/main.tf
resource "azurerm_resource_group" "compute" {
  name     = "rg-${var.environment}-compute"
  location = var.location
  tags     = var.tags
}

# App Service Plan
resource "azurerm_service_plan" "main" {
  name                = "asp-${var.environment}"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v3" : "B1"
  tags                = var.tags
}

# Linux Web App
resource "azurerm_linux_web_app" "api" {
  name                = "app-${var.environment}-api-${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_service_plan.main.location
  service_plan_id     = azurerm_service_plan.main.id
  tags                = var.tags

  site_config {
    always_on = var.environment == "prod" ? true : false
    
    application_stack {
      docker_image_name = "mcr.microsoft.com/dotnet/aspnet:8.0"
    }
  }

  identity {
    type = "UserAssigned"
    identity_ids = [var.managed_identity_id]
  }
}
```

### Container Module (Container Apps, AKS)
```hcl
# infra/app/modules/container-module/main.tf
resource "azurerm_resource_group" "container" {
  name     = "rg-${var.environment}-container"
  location = var.location
  tags     = var.tags
}

# Container Apps Environment
resource "azurerm_container_app_environment" "main" {
  name                       = "cae-${var.environment}"
  location                   = azurerm_resource_group.container.location
  resource_group_name        = azurerm_resource_group.container.name
  log_analytics_workspace_id = var.log_analytics_workspace_id
  tags                       = var.tags
}

# Container App
resource "azurerm_container_app" "api" {
  name                         = "ca-${var.environment}-api"
  container_app_environment_id = azurerm_container_app_environment.main.id
  resource_group_name          = azurerm_resource_group.container.name
  revision_mode                = "Single"
  tags                         = var.tags

  template {
    container {
      name   = "api"
      image  = "mcr.microsoft.com/dotnet/aspnet:8.0"
      cpu    = 0.25
      memory = "0.5Gi"
    }
  }

  ingress {
    external_enabled = true
    target_port      = 80
    traffic_weight {
      percentage      = 100
      latest_revision = true
    }
  }
}
```

### Database Module
```hcl
# infra/app/modules/database-module/main.tf
resource "azurerm_resource_group" "database" {
  name     = "rg-${var.environment}-database"
  location = var.location
  tags     = var.tags
}

# PostgreSQL Flexible Server
resource "azurerm_postgresql_flexible_server" "main" {
  name                   = "psql-${var.environment}-${random_string.suffix.result}"
  resource_group_name    = azurerm_resource_group.database.name
  location               = azurerm_resource_group.database.location
  version                = "15"
  administrator_login    = "psqladmin"
  administrator_password = random_password.db_password.result
  
  storage_mb   = var.environment == "prod" ? 32768 : 32768
  sku_name     = var.environment == "prod" ? "GP_Standard_D2s_v3" : "B_Standard_B1ms"
  
  tags = var.tags
}

resource "azurerm_postgresql_flexible_server_database" "main" {
  name      = var.database_name
  server_id = azurerm_postgresql_flexible_server.main.id
  collation = "en_US.utf8"
  charset   = "utf8"
}
```

## Publishing Modules to Terraform Cloud 📦

Once you've built your modules, you can publish them to Terraform Cloud's private registry for reuse across projects.

### Step 1: Organize Module Repository

Create a separate Git repository for each module:

```
terraform-azure-networking/
├── main.tf
├── variables.tf
├── outputs.tf
├── README.md
└── examples/
    └── complete/
        └── main.tf
```

### Step 2: Tag Your Module

Terraform Cloud uses Git tags for versioning:

```bash
cd terraform-azure-networking

# Create a version tag
git tag -a "v1.0.0" -m "Initial release"
git push origin v1.0.0
```

### Step 3: Connect to Terraform Cloud

In Terraform Cloud UI:

1. Navigate to **Registry** → **Publish** → **Module**
2. Select your VCS provider (GitHub)
3. Choose the repository (e.g., `terraform-azure-networking`)
4. Click **Publish Module**

### Step 4: Use Published Modules

Now reference your modules from the private registry:

```hcl
module "networking" {
  source  = "app.terraform.io/my-org/networking/azure"
  version = "~> 1.0"
  
  environment = var.environment
  location    = var.location
  tags        = var.tags
}
```

**Module Naming Convention**: `terraform-<PROVIDER>-<NAME>`
- `terraform-azure-networking`
- `terraform-azure-identity`
- `terraform-azure-integration`

### Step 5: Module Best Practices

**Complete Variables File**
```hcl
# variables.tf
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "eastus"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**Complete Outputs File**
```hcl
# outputs.tf
output "vnet_id" {
  description = "Virtual Network ID"
  value       = azurerm_virtual_network.main.id
}

output "subnet_ids" {
  description = "Map of subnet names to IDs"
  value = {
    app       = azurerm_subnet.app.id
    container = azurerm_subnet.container.id
    database  = azurerm_subnet.database.id
  }
}
```

**Comprehensive README**
```markdown
# Azure Networking Module

Provisions Azure Virtual Network with subnets for app, container, and database tiers.

## Usage

```hcl
module "networking" {
  source  = "app.terraform.io/my-org/networking/azure"
  version = "~> 1.0"
  
  environment = "dev"
  location    = "eastus"
  
  tags = {
    Project   = "Platform"
    ManagedBy = "Terraform"
  }
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| environment | Environment name | string | n/a | yes |
| location | Azure region | string | eastus | no |

## Outputs

| Name | Description |
|------|-------------|
| vnet_id | Virtual Network ID |
| subnet_ids | Map of subnet IDs |
```

## Workspace Management 🔄

### Creating Workspaces via CLI

You can create workspaces programmatically:

```bash
# Create workspace
cat > workspace-dev.json <<EOF
{
  "data": {
    "attributes": {
      "name": "app-dev",
      "resource-count": 0,
      "terraform-version": "1.6.0",
      "working-directory": "infra/app",
      "execution-mode": "remote",
      "vcs-repo": {
        "identifier": "my-org/platform-infrastructure",
        "branch": "main",
        "oauth-token-id": "ot-XXXXX"
      }
    },
    "type": "workspaces"
  }
}
EOF

curl \
  --header "Authorization: Bearer $TF_CLOUD_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @workspace-dev.json \
  https://app.terraform.io/api/v2/organizations/$TF_CLOUD_ORG/workspaces
```

### Workspace Variables via CLI

Set variables programmatically:

```bash
# Set environment variable
cat > variable.json <<EOF
{
  "data": {
    "type": "vars",
    "attributes": {
      "key": "ARM_CLIENT_ID",
      "value": "$CLIENT_ID",
      "category": "env",
      "hcl": false,
      "sensitive": false
    }
  }
}
EOF

curl \
  --header "Authorization: Bearer $TF_CLOUD_TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @variable.json \
  https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/vars
```

## Running Terraform 🚀

### Via Terraform Cloud UI

1. Navigate to workspace
2. Click **Actions** → **Start new run**
3. Add a message describing the change
4. Review the plan
5. Click **Confirm & Apply**

### Via CLI

```bash
cd infra/shared

# Initialize
terraform init

# Plan
terraform plan -out=tfplan

# Apply (triggers remote execution)
terraform apply tfplan
```

### Via VCS Integration

When connected to GitHub:

1. Push code to your repository
2. Terraform Cloud automatically triggers a plan
3. Review the plan in a PR comment
4. Merge PR to trigger apply (if auto-apply enabled)

## Complete Example 🎯

Let's put it all together with a complete example:

**Directory Structure**
```
platform-infrastructure/
├── infra/
│   ├── shared/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tf
│   │   └── modules/
│   │       ├── identity-module/
│   │       ├── networking-module/
│   │       ├── integration-module/
│   │       └── storage-module/
│   └── app/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tf
│       └── modules/
│           ├── compute-module/
│           ├── container-module/
│           └── database-module/
└── .github/
    └── workflows/
        └── terraform-ci.yml
```

**GitHub Actions Workflow** (`.github/workflows/terraform-ci.yml`)
```yaml
name: Terraform CI

on:
  pull_request:
    paths:
      - 'infra/**'
  push:
    branches:
      - main
    paths:
      - 'infra/**'

jobs:
  terraform:
    name: Terraform Plan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
      
      - name: Terraform Format
        run: terraform fmt -check -recursive infra/
      
      - name: Terraform Init - Shared
        run: |
          cd infra/shared
          terraform init
      
      - name: Terraform Validate - Shared
        run: |
          cd infra/shared
          terraform validate
      
      - name: Terraform Init - App
        run: |
          cd infra/app
          terraform init
      
      - name: Terraform Validate - App
        run: |
          cd infra/app
          terraform validate
```

## Benefits of This Approach 🎁

**Security**
- ✅ No stored secrets - OIDC authentication
- ✅ Centralized credential management
- ✅ Audit logging

**Collaboration**
- ✅ Team state management without conflicts
- ✅ VCS integration for code review
- ✅ Automated planning on PRs

**Standardization**
- ✅ Reusable modules based on Azure patterns
- ✅ Consistent resource naming
- ✅ Private registry for sharing

**Scalability**
- ✅ Separate workspaces per environment
- ✅ Parallel execution across projects
- ✅ Cost tracking per workspace

## Common Patterns 📋

### Environment-Specific Configurations

Use workspace variables for environment-specific values:

```hcl
# In Terraform Cloud, set these as Terraform variables:
# workspace: app-dev
environment = "dev"
location    = "eastus"
sku_tier    = "Basic"

# workspace: app-prod
environment = "prod"
location    = "eastus2"
sku_tier    = "Standard"
```

### Cross-Environment Data Sharing

Use `terraform_remote_state` to share data:

```hcl
# App infrastructure needs networking info from shared
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-services-${var.environment}"
    }
  }
}

# Use shared outputs
subnet_id = data.terraform_remote_state.shared.outputs.app_subnet_id
```

### Module Composition

Build higher-level modules from lower-level ones:

```hcl
# infra/shared/modules/complete-platform/main.tf
module "networking" {
  source = "app.terraform.io/my-org/networking/azure"
  version = "~> 1.0"
  # ...
}

module "identity" {
  source = "app.terraform.io/my-org/identity/azure"
  version = "~> 1.0"
  # ...
}

module "integration" {
  source = "app.terraform.io/my-org/integration/azure"
  version = "~> 1.0"
  vnet_id = module.networking.vnet_id
  # ...
}
```

## Troubleshooting 🔧

### OIDC Authentication Issues

If you get authentication errors:

```bash
# Verify service principal exists
az ad sp show --id $CLIENT_ID

# Check federated credential
az ad app federated-credential list --id $CLIENT_ID

# Verify workspace variables
# In Terraform Cloud UI, check:
# - ARM_CLIENT_ID is set
# - ARM_SUBSCRIPTION_ID is set
# - ARM_TENANT_ID is set
# - ARM_USE_OIDC is "true"
```

### State Locking Issues

If state is locked:

```bash
# View lock info in Terraform Cloud UI
# Or force unlock (use with caution!)
terraform force-unlock <LOCK_ID>
```

### Module Not Found

If modules aren't found:

```bash
# Ensure module is published with proper tag
git tag -l

# Check module name format: terraform-<PROVIDER>-<NAME>
# Check version constraint in module source
```

## Next Steps 🚀

We've now set up:
- ✅ Terraform Cloud with secure Azure authentication
- ✅ Organized infrastructure code (shared vs. app)
- ✅ Reusable modules based on Azure Charts patterns
- ✅ Private module registry

In the next post, we'll explore **Azure DevOps Pipelines** to automate infrastructure deployment and integrate with our application CI/CD workflows.

## Resources 📚

- [Terraform Cloud Documentation](https://developer.hashicorp.com/terraform/cloud-docs)
- [Azure Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure Charts Resource Classification](https://azurecharts.com/)
- [Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation)
- [Module Registry Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/registry/publish-modules)

---

Questions? Want to discuss Terraform patterns? Find me on [LinkedIn](https://linkedin.com/in/brianpsheridan) or [GitHub](https://github.com/two4suited)!

