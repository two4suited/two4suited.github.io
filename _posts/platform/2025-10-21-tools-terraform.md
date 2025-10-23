---
title: "Building a Modern Development Platform: Terraform & Terraform Cloud for Azure Infrastructure 🏗️"
date: 2025-10-21T05:00:00-07:00
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

**📦 Code Repository**: All the Terraform code from this tutorial is available on GitHub at [blog-platform-aspire/aspire-tools-terraform](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-terraform).

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
- CLI-driven runs for infrastructure deployments
- VCS integration for module versioning and publishing
- Shared state management across teams
- Private module registry for code reuse

**Cost Optimization**
- Free tier supports up to 500 resources per month
- Pay only for what you use beyond that
- **Note**: While 500 free resources sounds generous, the per-resource cost beyond that is quite high. If you're managing large-scale infrastructure across multiple environments, costs can add up quickly. In a future post, we'll explore migrating state management to Azure Storage Account as a more cost-effective alternative for teams with extensive infrastructure footprints.

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
├── Project: Shared Services
│   ├── Workspace: shared-services        # Cross-environment resources
│                                         # (App Config, Key Vault, ACR)
├── Project: Environment Infrastructure
│   ├── Workspace: env-dev               # Environment-level resources
│   ├── Workspace: env-staging           # (Front Door, APIM, App Service Plans)
│   └── Workspace: env-prod
└── Project: Applications
    ├── Workspace: app-weather-dev       # App-specific resources
    ├── Workspace: app-weather-staging   # (Web Apps, settings, FD/APIM config)
    └── Workspace: app-weather-prod
```

**Note**: We'll dive deeper into this Azure resource organization strategy in a later post on Azure environment architecture.

### Creating Projects

In Terraform Cloud UI:

1. Navigate to **Projects** → **New Project**
2. Create a project called **"platform-blog-post"**
3. We'll create workspaces within this project in the next steps

For this tutorial, we'll keep it simple with one project and three workspaces:
- **shared-services** - For cross-environment shared resources (ACR)
- **env-test** - For environment-level infrastructure (App Service Plan)
- **app-test** - For application-specific resources (Web App)

**Creating Workspaces:**

1. In Terraform Cloud, navigate to your **platform-blog-post** project
2. Click **New workspace**
3. Choose **CLI-driven workflow** (not VCS-driven)
4. Name it **shared-services** (repeat for **env-test** and **app-test**)

You should end up with three workspaces in your project:
- `shared-services` - Deploy first (no dependencies)
- `env-test` - Deploy second (depends on shared-services)
- `app-test` - Deploy third (depends on both shared-services and env-test)

**Why CLI-driven workflow?**
- Gives you full control over when and how infrastructure is deployed
- Allows orchestration via pipelines (GitHub Actions, Azure DevOps, etc.)
- Perfect for complex deployment workflows with dependencies
- You trigger runs from your local machine or CI/CD pipeline, not from Git commits

We'll explore pipeline automation in a future post on Azure DevOps Pipelines.

### Installing Terraform CLI

Before we can work with Terraform Cloud, we need to install the Terraform CLI:

**macOS (using Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Verify installation
terraform version
```

**Windows (using Chocolatey):**
```bash
choco install terraform

# Verify installation
terraform version
```

**Linux (Ubuntu/Debian):**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Verify installation
terraform version
```

### Logging into Terraform Cloud

Once Terraform is installed, authenticate with Terraform Cloud:

```bash
# Login to Terraform Cloud
terraform login

# This will:
# 1. Open your browser to https://app.terraform.io/app/settings/tokens
# 2. Prompt you to create an API token
# 3. Copy the token back to your terminal
# 4. Store the token in ~/.terraform.d/credentials.tfrc.json
```

**What happens during login:**
- Terraform opens your browser to the token creation page
- You create a token (give it a descriptive name like "Local Development")
- The token is saved locally and used for all future Terraform Cloud API calls
- This token authenticates your CLI operations (plan, apply, etc.)

**Manual token creation (alternative):**
If `terraform login` doesn't work, you can manually create the credentials file:

```bash
# Create the credentials file
mkdir -p ~/.terraform.d
cat > ~/.terraform.d/credentials.tfrc.json <<EOF
{
  "credentials": {
    "app.terraform.io": {
      "token": "YOUR_TOKEN_HERE"
    }
  }
}
EOF
```

## Azure Service Principal Setup 🔐

Now comes the critical part: setting up secure authentication between Terraform Cloud and Azure. We'll use **Workload Identity Federation** instead of storing client secrets.

**Prerequisites:**
- Azure CLI installed ([installation guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli))
- An Azure subscription

### Step 1: Create the Service Principal

First, authenticate with Azure:

```bash
# Login to Azure
az login

# If you have multiple subscriptions, list them
az account list --output table

# Set the subscription you want to use
SUBSCRIPTION_ID="your-subscription-id"
az account set --subscription $SUBSCRIPTION_ID

# Verify you're using the correct subscription
az account show
```

Now create the service principal:

```bash
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

**Important**: Azure federated credentials don't support wildcards (`*`) in the subject pattern. You need to create a separate credential for each workspace and run phase combination.

```bash
# Get your Terraform Cloud organization name
TF_CLOUD_ORG="my-org"

# IMPORTANT: Project name must match EXACTLY as shown in Terraform Cloud (case-sensitive)
PROJECT_NAME="platform-blog-post"

# For our three-tier architecture, we need credentials for each workspace's plan and apply phases
# That's 6 total credentials: 3 workspaces × 2 run phases (plan + apply)

# 1. Shared Services - Plan
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-shared-services-plan",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:shared-services:run_phase:plan",
    "description": "Terraform Cloud - shared-services workspace, plan phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 2. Shared Services - Apply
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-shared-services-apply",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:shared-services:run_phase:apply",
    "description": "Terraform Cloud - shared-services workspace, apply phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 3. Environment Test - Plan
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-env-test-plan",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:env-test:run_phase:plan",
    "description": "Terraform Cloud - env-test workspace, plan phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Environment Test - Apply
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-env-test-apply",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:env-test:run_phase:apply",
    "description": "Terraform Cloud - env-test workspace, apply phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 5. App Test - Plan
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-app-test-plan",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:app-test:run_phase:plan",
    "description": "Terraform Cloud - app-test workspace, plan phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 6. App Test - Apply
az ad app federated-credential create \
  --id $CLIENT_ID \
  --parameters '{
    "name": "tfc-app-test-apply",
    "issuer": "https://app.terraform.io",
    "subject": "organization:'$TF_CLOUD_ORG':project:'$PROJECT_NAME':workspace:app-test:run_phase:apply",
    "description": "Terraform Cloud - app-test workspace, apply phase",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**Understanding the subject pattern:**

The `subject` field defines which Terraform Cloud runs can use this credential:

```
organization:<ORG>:project:<PROJECT>:workspace:<WORKSPACE>:run_phase:<PHASE>
```

- `organization`: Your Terraform Cloud org name (required, exact match)
- `project`: Project name (exact match, no wildcards)
- `workspace`: Workspace name (exact match, no wildcards)
- `run_phase`: Either `plan` or `apply` (exact match, no wildcards)

**Important**: Wildcards (`*`) are **not supported** in Azure federated credentials. Each workspace and run phase combination requires its own credential.

**Security best practices:**

- **Development**: Create credentials for both `plan` and `apply` phases per workspace
- **Production**: Consider creating only `apply` credentials to prevent unauthorized planning (though both are typically needed)
- **Multiple environments**: Create separate service principals for dev, staging, and prod with appropriate Azure RBAC permissions
- **Least privilege**: Each service principal should only have access to resources in its environment

**Example: Separate service principals for dev and prod:**

```bash
# Dev service principal - for dev/test workspaces
SP_NAME_DEV="terraform-platform-dev-sp"
az ad sp create-for-rbac \
  --name $SP_NAME_DEV \
  --role Contributor \
  --scopes "/subscriptions/$DEV_SUBSCRIPTION_ID"

CLIENT_ID_DEV=$(az ad sp list --display-name $SP_NAME_DEV --query "[0].appId" -o tsv)

# Create credentials for dev workspaces
az ad app federated-credential create \
  --id $CLIENT_ID_DEV \
  --parameters '{
    "name": "tfc-env-dev-plan",
    "subject": "organization:'$TF_CLOUD_ORG':project:platform-blog-post:workspace:env-dev:run_phase:plan",
    "audiences": ["api://AzureADTokenExchange"]
  }'

az ad app federated-credential create \
  --id $CLIENT_ID_DEV \
  --parameters '{
    "name": "tfc-env-dev-apply",
    "subject": "organization:'$TF_CLOUD_ORG':project:platform-blog-post:workspace:env-dev:run_phase:apply",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# Prod service principal - for prod workspaces only
SP_NAME_PROD="terraform-platform-prod-sp"
az ad sp create-for-rbac \
  --name $SP_NAME_PROD \
  --role Contributor \
  --scopes "/subscriptions/$PROD_SUBSCRIPTION_ID"

CLIENT_ID_PROD=$(az ad sp list --display-name $SP_NAME_PROD --query "[0].appId" -o tsv)

# Create credentials for prod workspaces
az ad app federated-credential create \
  --id $CLIENT_ID_PROD \
  --parameters '{
    "name": "tfc-env-prod-plan",
    "subject": "organization:'$TF_CLOUD_ORG':project:platform-blog-post:workspace:env-prod:run_phase:plan",
    "audiences": ["api://AzureADTokenExchange"]
  }'

az ad app federated-credential create \
  --id $CLIENT_ID_PROD \
  --parameters '{
    "name": "tfc-env-prod-apply",
    "subject": "organization:'$TF_CLOUD_ORG':project:platform-blog-post:workspace:env-prod:run_phase:apply",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

**What's happening here?**

- Terraform Cloud requests a token from Azure AD when runs execute
- Azure validates the token matches the exact subject pattern (org/project/workspace/phase)
- No client secret needed - the trust is based on the issuer (Terraform Cloud) and the specific subject pattern
- Each workspace/phase combination needs its own federated credential (6 total for our 3 workspaces)
- Azure enforces exact matching - no wildcards are supported in the subject field

**Credential naming convention:**

Use descriptive names for easy management: `tfc-<workspace>-<phase>`
- `tfc-shared-services-plan`
- `tfc-shared-services-apply`
- `tfc-env-test-plan`
- `tfc-env-test-apply`
- `tfc-app-test-plan`
- `tfc-app-test-apply`

**Scaling to multiple environments:**

When you add more environments (dev, staging, prod), you'll create additional credentials:
- For `env-dev`, `env-staging`, `env-prod` workspaces: 6 more credentials (3 workspaces × 2 phases)
- For `app-myapp-dev`, `app-myapp-staging`, `app-myapp-prod`: 6 more credentials per app
- Consider using separate service principals per environment for better security isolation

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

### Step 4: Configure Terraform Cloud Variable Set

Instead of configuring variables individually for each workspace, we'll create a **Variable Set** and apply it to the entire project. This makes it easy to share authentication credentials across all workspaces.

**In Terraform Cloud UI:**

1. Navigate to **Settings** → **Variable Sets** → **Create variable set**
2. Name it **"Azure Authentication - Dev"** (or your environment name)
3. Add a description: "Azure service principal credentials for OIDC authentication"
4. Add the following **Environment Variables**:

```hcl
# Azure Authentication (Environment Variables)
ARM_SUBSCRIPTION_ID         = "<subscription-id>"
ARM_TENANT_ID              = "<tenant-id>"         # From step 1
TFC_AZURE_PROVIDER_AUTH    = "true"               # Enable OIDC for Terraform Cloud
TFC_AZURE_RUN_CLIENT_ID    = "<client-id>"        # From step 1
```

5. Apply the variable set to:
   - **Scope**: Select "Apply to specific projects and workspaces"
   - **Projects**: Choose your **"platform-blog-post"** project
   - This automatically applies to ALL workspaces in the project (env-test, app-test, etc.)

**Important notes:**
- Mark these as **Environment Variables**, NOT Terraform variables
- You can mark `ARM_SUBSCRIPTION_ID`, `ARM_TENANT_ID`, and `TFC_AZURE_RUN_CLIENT_ID` as sensitive if desired
- `TFC_AZURE_PROVIDER_AUTH` is the Terraform Cloud-specific variable that enables OIDC authentication
- `TFC_AZURE_RUN_CLIENT_ID` is the client ID of the Azure AD application/service principal

**Benefits of Variable Sets:**
- ✅ Configure once, apply to all workspaces in the project
- ✅ Easy to update credentials across all workspaces
- ✅ Can create separate variable sets for different environments (dev/staging/prod)
- ✅ Can scope to specific projects or workspaces as needed

**For multiple environments:**

You might create separate variable sets for different environments:
- **"Azure Authentication - Dev"** → Applied to dev-related projects
- **"Azure Authentication - Staging"** → Applied to staging projects  
- **"Azure Authentication - Prod"** → Applied to prod projects (with different service principal)

Each variable set would use a different `TFC_AZURE_RUN_CLIENT_ID` corresponding to the service principal created for that environment (with appropriate subject pattern scoping).

No `ARM_CLIENT_SECRET` needed! 🎉

## Infrastructure Code Organization 📁

Now let's structure our Terraform code using a three-tier architecture that separates concerns by lifecycle and scope. For this tutorial, we'll focus on the essential building blocks: **Container Registry**, **App Service Plan**, and **Web App**.

**Note**: For this tutorial, we're keeping modules local within each infrastructure layer. In a future post, we'll extract these modules into separate Git repositories and publish them to Terraform Cloud's private registry for reuse across multiple projects.

**File Organization**: While Terraform allows you to put all configuration in a single `.tf` file (Terraform loads all `.tf` files in a directory), I prefer breaking it out into separate files (`main.tf`, `variables.tf`, `outputs.tf`, `terraform.tf`) for better organization and readability. This separation makes it easier to find specific configurations and follows common Terraform conventions.

We'll create an `infra` folder with three main layers:

```
infra/
├── shared-services/             # Cross-environment shared resources
│   ├── main.tf                 # Resource definitions
│   ├── variables.tf            # Input variables
│   ├── outputs.tf              # Output values
│   ├── terraform.tf            # Terraform & provider config
│   └── modules/
│       └── container-registry/  # Azure Container Registry
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
│
├── environment/                 # Environment-level infrastructure
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tf
│   └── modules/
│       └── app-service-plan/    # Shared App Service Plans
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
│
└── app/                         # Application-specific resources
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── terraform.tf
    └── modules/
        └── web-app/             # App Service Web Apps
            ├── main.tf
            ├── variables.tf
            └── outputs.tf
```

**Three-tier structure explained:**

1. **Shared Services** (`infra/shared-services/`)
   - Resources used across **all environments** (dev, staging, prod)
   - Deployed **once** and referenced by all other layers
   - Example: Container Registry for storing Docker images

2. **Environment** (`infra/environment/`)
   - Resources specific to an **environment** but shared across apps
   - Deployed **per environment** (separate for dev, staging, prod)
   - Example: App Service Plan (shared compute for multiple apps)

3. **App** (`infra/app/`)
   - Resources for a **specific application**
   - Deployed **per app per environment**
   - Example: Web App instance with app-specific settings

**Why this structure?**

- **Clear separation of concerns** - Different lifecycles for different resource types
- **Resource sharing** - ACR shared across all environments, App Service Plan shared within an environment
- **Independent deployments** - Can update app without touching shared services
- **Cost optimization** - Share expensive resources (ACR, App Service Plans) where appropriate

Later, we can expand each layer with additional modules like Key Vault, Front Door, API Management, and Application Insights.

**Note**: We'll dive deeper into this Azure resource organization strategy in our upcoming post on Azure environment architecture.

### Layer 1: Shared Services Infrastructure

This contains resources shared across **all environments** - deployed once and used by dev, staging, and prod.

**`infra/shared-services/terraform.tf`**
```hcl
terraform {
  required_version = ">= 1.6"

  cloud {
    organization = "my-org"
    
    workspaces {
      name = "shared-services"
    }
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.40"
    }
  }
}

provider "azurerm" {
  features {}
  
  # OIDC authentication - automatically detected from TFC_AZURE_PROVIDER_AUTH environment variable
}
```

**`infra/shared-services/variables.tf`**
```hcl
variable "location" {
  description = "Azure region for shared resources"
  type        = string
  default     = "westus2"
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Project   = "Platform"
    Layer     = "Shared-Services"
  }
}
```

**`infra/shared-services/main.tf`**
```hcl
# Resource Group for shared services
resource "azurerm_resource_group" "shared" {
  name     = "rg-shared-services"
  location = var.location
  tags     = var.tags
}

# Azure Container Registry - shared across all environments
module "container_registry" {
  source = "./modules/container-registry"
  
  resource_group_name = azurerm_resource_group.shared.name
  location            = azurerm_resource_group.shared.location
  tags                = var.tags
}
```

**`infra/shared-services/outputs.tf`**
```hcl
output "resource_group_name" {
  description = "Shared services resource group name"
  value       = azurerm_resource_group.shared.name
}

output "acr_id" {
  description = "Container Registry ID"
  value       = module.container_registry.id
}

output "acr_login_server" {
  description = "Container Registry login server"
  value       = module.container_registry.login_server
}

output "acr_name" {
  description = "Container Registry name"
  value       = module.container_registry.name
}
```

**Container Registry Module** (`infra/shared-services/modules/container-registry/main.tf`)
```hcl
resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}

resource "azurerm_container_registry" "main" {
  name                = "acrshared${random_string.suffix.result}"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Basic"
  admin_enabled       = false
  
  tags = var.tags
}
```

**Container Registry Module Variables** (`infra/shared-services/modules/container-registry/variables.tf`)
```hcl
variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

**Container Registry Module Outputs** (`infra/shared-services/modules/container-registry/outputs.tf`)
```hcl
output "id" {
  description = "Container Registry ID"
  value       = azurerm_container_registry.main.id
}

output "login_server" {
  description = "Container Registry login server"
  value       = azurerm_container_registry.main.login_server
}

output "name" {
  description = "Container Registry name"
  value       = azurerm_container_registry.main.name
}
```

**Deploying Shared Services**

Now deploy the shared services layer:

```bash
cd infra/shared-services
terraform init
terraform plan
terraform apply  # Add --auto-approve to skip confirmation prompt

# Capture outputs for later use
ACR_LOGIN_SERVER=$(terraform output -raw acr_login_server)
echo "ACR Login Server: $ACR_LOGIN_SERVER"
```

### Layer 2: Environment Infrastructure

This contains resources specific to an **environment** but shared across applications within that environment.

**`infra/environment/terraform.tf`**
```hcl
terraform {
  required_version = ">= 1.6"

  cloud {
    organization = "my-org"
    
    workspaces {
      name = "env-test"  # For tutorial; use env-dev, env-staging, env-prod for multiple environments
    }
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.40"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**`infra/environment/variables.tf`**
```hcl
variable "environment" {
  description = "Environment name (test, dev, staging, prod)"
  type        = string
  
  validation {
    condition     = contains(["test", "dev", "staging", "prod"], var.environment)
    error_message = "Environment must be test, dev, staging, or prod."
  }
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "westus2"
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Project   = "Platform"
  }
}
```

**`infra/environment/main.tf`**
```hcl
# Reference shared services
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-services"
    }
  }
}

# Resource Group for environment infrastructure
resource "azurerm_resource_group" "environment" {
  name     = "rg-${var.environment}"
  location = var.location
  tags     = merge(var.tags, { Environment = var.environment })
}

# App Service Plan - shared compute for all apps in this environment
module "app_service_plan" {
  source = "./modules/app-service-plan"
  
  resource_group_name = azurerm_resource_group.environment.name
  environment         = var.environment
  location            = azurerm_resource_group.environment.location
  tags                = merge(var.tags, { Environment = var.environment })
}
```

**`infra/environment/outputs.tf`**
```hcl
output "app_service_plan_id" {
  description = "App Service Plan ID"
  value       = module.app_service_plan.id
}

output "app_service_plan_name" {
  description = "App Service Plan name"
  value       = module.app_service_plan.name
}

# Pass through shared services outputs for convenience
output "acr_login_server" {
  description = "Container Registry login server from shared services"
  value       = data.terraform_remote_state.shared.outputs.acr_login_server
}
```

**App Service Plan Module** (`infra/environment/modules/app-service-plan/main.tf`)
```hcl
resource "azurerm_service_plan" "main" {
  name                = "asp-${var.environment}"
  resource_group_name = var.resource_group_name
  location            = var.location
  os_type             = "Linux"
  sku_name            = var.environment == "prod" ? "P1v3" : "P0v3"
  
  tags = var.tags
}
```

**App Service Plan Module Variables** (`infra/environment/modules/app-service-plan/variables.tf`)
```hcl
variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

**App Service Plan Module Outputs** (`infra/environment/modules/app-service-plan/outputs.tf`)
```hcl
output "id" {
  description = "App Service Plan ID"
  value       = azurerm_service_plan.main.id
}

output "name" {
  description = "App Service Plan name"
  value       = azurerm_service_plan.main.name
}
```

**Deploying Environment Infrastructure**

Deploy the environment layer (requires shared services to be deployed first):

```bash
cd ../environment
terraform init
terraform plan -var="environment=test"
terraform apply -var="environment=test"  # Add --auto-approve to skip confirmation prompt

# Capture outputs
APP_SERVICE_PLAN_ID=$(terraform output -raw app_service_plan_id)
echo "App Service Plan ID: $APP_SERVICE_PLAN_ID"
```

### Layer 3: Application Infrastructure

This contains resources for a **specific application** - deployed per app per environment.

**`infra/app/terraform.tf`**
```hcl
terraform {
  required_version = ">= 1.6"

  cloud {
    organization = "my-org"
    
    workspaces {
      name = "app-test"  # For tutorial; use app-myapp-dev, app-myapp-staging, app-myapp-prod for multiple environments
    }
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.40"
    }
  }
}

provider "azurerm" {
  features {}
}
```

**`infra/app/variables.tf`**
```hcl
variable "environment" {
  description = "Environment name (test, dev, staging, prod)"
  type        = string
  
  validation {
    condition     = contains(["test", "dev", "staging", "prod"], var.environment)
    error_message = "Environment must be test, dev, staging, or prod."
  }
}

variable "app_name" {
  description = "Application name"
  type        = string
  default     = "myapp"
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "westus2"
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Project   = "Platform"
  }
}
```

**`infra/app/main.tf`**
```hcl
# Reference shared services and environment infrastructure
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-services"
    }
  }
}

data "terraform_remote_state" "environment" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "env-${var.environment}"
    }
  }
}

# Resource Group for application
resource "azurerm_resource_group" "app" {
  name     = "rg-${var.environment}-${var.app_name}"
  location = var.location
  tags = merge(var.tags, { 
    Environment = var.environment
    Application = var.app_name
  })
}

# Web App
module "web_app" {
  source = "./modules/web-app"
  
  resource_group_name = azurerm_resource_group.app.name
  app_name            = var.app_name
  environment         = var.environment
  location            = azurerm_resource_group.app.location
  service_plan_id     = data.terraform_remote_state.environment.outputs.app_service_plan_id
  acr_login_server    = data.terraform_remote_state.shared.outputs.acr_login_server
  tags                = merge(var.tags, { 
    Environment = var.environment
    Application = var.app_name
  })
}
```

**`infra/app/outputs.tf`**
```hcl
output "web_app_name" {
  description = "Web App name"
  value       = module.web_app.name
}

output "web_app_url" {
  description = "Web App URL"
  value       = module.web_app.url
}

output "web_app_default_hostname" {
  description = "Web App default hostname"
  value       = module.web_app.default_hostname
}
```

**Web App Module** (`infra/app/modules/web-app/main.tf`)
```hcl
resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}

resource "azurerm_linux_web_app" "main" {
  name                = "app-${var.environment}-${var.app_name}-${random_string.suffix.result}"
  resource_group_name = var.resource_group_name
  location            = var.location
  service_plan_id     = var.service_plan_id
  
  site_config {
    always_on = var.environment == "prod" ? true : false
    
    application_stack {
      docker_image_name   = "${var.acr_login_server}/${var.app_name}:latest"
      docker_registry_url = "https://${var.acr_login_server}"
    }
  }

  identity {
    type = "SystemAssigned"
  }
  
  tags = var.tags
}

# Get ACR ID for role assignment
data "azurerm_container_registry" "acr" {
  name                = split(".", var.acr_login_server)[0]
  resource_group_name = "rg-shared-services"
}

# Grant Web App access to pull images from ACR
resource "azurerm_role_assignment" "acr_pull" {
  principal_id                     = azurerm_linux_web_app.main.identity[0].principal_id
  role_definition_name             = "AcrPull"
  scope                            = data.azurerm_container_registry.acr.id
  skip_service_principal_aad_check = true
}
```

**Web App Module Variables** (`infra/app/modules/web-app/variables.tf`)
```hcl
variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "app_name" {
  description = "Application name"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "service_plan_id" {
  description = "App Service Plan ID"
  type        = string
}

variable "acr_login_server" {
  description = "Container Registry login server"
  type        = string
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

**Web App Module Outputs** (`infra/app/modules/web-app/outputs.tf`)
```hcl
output "id" {
  description = "Web App ID"
  value       = azurerm_linux_web_app.main.id
}

output "name" {
  description = "Web App name"
  value       = azurerm_linux_web_app.main.name
}

output "default_hostname" {
  description = "Web App default hostname"
  value       = azurerm_linux_web_app.main.default_hostname
}

output "url" {
  description = "Web App URL"
  value       = "https://${azurerm_linux_web_app.main.default_hostname}"
}
```

**Deploying Application Infrastructure**

Deploy the application layer (requires both shared services and environment to be deployed):

```bash
cd ../app
terraform init
terraform plan -var="environment=test" -var="app_name=myapp"
terraform apply -var="environment=test" -var="app_name=myapp"  # Add --auto-approve to skip confirmation prompt

# Get the app URL
APP_URL=$(terraform output -raw web_app_url)
echo "App deployed at: $APP_URL"
```

## Common Patterns 📋

### Environment-Specific Configurations

Use workspace variables for environment-specific values:

```hcl
# In Terraform Cloud, set these as Terraform variables:
# workspace: app-dev
environment = "dev"
location    = "westus2"
sku_tier    = "Basic"

# workspace: app-prod
environment = "prod"
location    = "westus2"
sku_tier    = "Standard"
```

### Cross-Workspace Data Sharing

Use `terraform_remote_state` to reference outputs from other workspaces:

```hcl
# App infrastructure needs data from shared services and environment
data "terraform_remote_state" "shared" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "shared-services"
    }
  }
}

data "terraform_remote_state" "environment" {
  backend = "remote"
  
  config = {
    organization = "my-org"
    workspaces = {
      name = "env-${var.environment}"
    }
  }
}

# Use outputs from other workspaces
acr_id           = data.terraform_remote_state.shared.outputs.acr_id
app_service_plan = data.terraform_remote_state.environment.outputs.app_service_plan_id
front_door_id    = data.terraform_remote_state.environment.outputs.front_door_id
```

This pattern creates clear dependencies:
- **Apps** depend on **Environment** and **Shared Services**
- **Environment** depends on **Shared Services**
- **Shared Services** has no dependencies (deployed first)

## Troubleshooting 🔧

### Missing Variable Configuration

If you get this error during `terraform plan`:

```
Error: unable to build authorizer for Resource Manager API: could not configure AzureCli Authorizer: 
could not parse Azure CLI version: launching Azure CLI: exec: "az": executable file not found in $PATH
```

**This means your Terraform Cloud environment variables are not configured correctly.** Terraform is trying to fall back to Azure CLI authentication because it can't find the OIDC credentials.

**Fix:**
1. Go to Terraform Cloud → **Settings** → **Variable Sets**
2. Verify your variable set has these **Environment Variables** (not Terraform variables):
   - `ARM_SUBSCRIPTION_ID` = Your Azure subscription ID
   - `ARM_TENANT_ID` = Your tenant ID
   - `TFC_AZURE_PROVIDER_AUTH` = `"true"` (as a string)
   - `TFC_AZURE_RUN_CLIENT_ID` = Your service principal client ID
3. Ensure the variable set is applied to your project or workspace
4. Run `terraform plan` again

**Note**: These must be **Environment Variables**, not Terraform variables. The category must be set to "env" in Terraform Cloud.

### Remote State Access Denied

If you get this error when trying to access remote state from another workspace:

```
Error: Error loading state: state data in S3 does not have the expected content.

This Terraform run is not authorized to read the state of the workspace 'shared-services'.
Most commonly, this is required when using the terraform_remote_state data source.
To allow this access, 'shared-services' must configure this workspace ('env-test')
as an authorized remote state consumer.
```

**This means the workspace you're trying to read state from hasn't granted your workspace permission to access its state.**

**Fix:**

1. Go to Terraform Cloud → Navigate to the **source workspace** (e.g., `shared-services`)
2. Go to **Settings** → **General**
3. Scroll down to **Remote state sharing**
4. Select **Share with specific workspaces**
5. Add the workspace(s) that need access (e.g., `env-test`, `app-test`)
6. Click **Save settings**
7. Run `terraform plan` again in your consuming workspace

**For our three-tier architecture, configure these permissions:**

- **shared-services** workspace: Grant access to `env-test` and `app-test`
- **env-test** workspace: Grant access to `app-test`

**Example configuration:**
```
Workspace: shared-services
├─ Remote state sharing: Share with specific workspaces
├─ Allowed workspaces:
│  ├─ env-test      ✓
│  └─ app-test      ✓

Workspace: env-test
├─ Remote state sharing: Share with specific workspaces
├─ Allowed workspaces:
│  └─ app-test      ✓
```

**Alternative: Global sharing (not recommended for production):**

You can also choose **Share with all workspaces in this organization**, but this is less secure as it allows any workspace to read the state. Use specific workspace sharing for better security.

### OIDC Authentication Issues

If you get authentication errors:

```bash
# Verify service principal exists
az ad sp show --id $CLIENT_ID

# Check federated credential
az ad app federated-credential list --id $CLIENT_ID

# Verify workspace variables
# In Terraform Cloud UI, check:
# - ARM_SUBSCRIPTION_ID is set
# - ARM_TENANT_ID is set
# - TFC_AZURE_PROVIDER_AUTH is "true"
# - TFC_AZURE_RUN_CLIENT_ID is set
```

### State Locking Issues

If state is locked:

```bash
# View lock info in Terraform Cloud UI
# Or force unlock (use with caution!)
terraform force-unlock <LOCK_ID>
```

## Cleaning Up Resources 🧹

**Important**: If you're just testing this setup and don't want to incur ongoing Azure charges, make sure to destroy the infrastructure when you're done. Destroy in **reverse order** of deployment:

```bash
# Step 1: Destroy application infrastructure first
cd infra/app
terraform destroy -var="environment=test" -var="app_name=myapp" --auto-approve

# Step 2: Destroy environment infrastructure
cd ../environment
terraform destroy -var="environment=test" --auto-approve

# Step 3: Destroy shared services last
cd ../shared-services
terraform destroy --auto-approve
```

**Why destroy in reverse order?**
- The app layer depends on the environment layer (App Service Plan)
- The environment layer depends on shared services (Container Registry)
- Destroying in reverse ensures dependencies are removed before their dependencies
- Terraform will error if you try to destroy a resource that's still being referenced

**Cost considerations:**
- **Container Registry (Basic SKU)**: ~$5/month
- **App Service Plan (P0v3)**: ~$58/month
- **Web App**: Included with App Service Plan

Total estimated cost for this tutorial setup: ~$63/month if left running.

## Next Steps 🚀

We've now set up:
- ✅ Terraform Cloud with secure Azure authentication
- ✅ Three-tier infrastructure organization (shared-services, environment, app)
- ✅ CLI-driven workflows for better developer experience

**Coming up in the series:**
- **Publishing Terraform Modules** - Extracting our local modules into separate Git repositories and publishing them to Terraform Cloud's private registry for reuse across projects
- **Complete Platform Infrastructure** - Building out the full infrastructure with all the modules (Key Vault, Front Door, API Management, Application Insights, and more) using our published modules
- **Azure Environment Architecture** - Deep dive into our three-tier resource organization strategy (shared services, environment resources, and application workspaces) and why this pattern works well for multi-environment platforms
- **Azure DevOps Pipelines** - Automating infrastructure deployment and application CI/CD workflows

## Resources 📚

- [Terraform Cloud Documentation](https://developer.hashicorp.com/terraform/cloud-docs)
- [Azure Provider Documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure Charts Resource Classification](https://azurecharts.com/)
- [Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/develop/workload-identity-federation)
- [Module Registry Best Practices](https://developer.hashicorp.com/terraform/cloud-docs/registry/publish-modules)

---

Questions? Want to discuss Terraform patterns? Find me on [LinkedIn](https://linkedin.com/in/brianpsheridan) or [GitHub](https://github.com/two4suited)!

