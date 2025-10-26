---
title: "Building a Modern Development Platform: Deploying Platform Documentation with Azure Storage and Front Door 📚"
date: 2025-10-28T06:00:00-07:00
draft: false
categories: ["platform","azure","terraform","static-sites","documentation","front-door"]
description: "Deploying platform documentation sites to Azure Storage with Front Door CDN using TechDocs, Azure DevOps pipelines, and Terraform Cloud for infrastructure as code"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction

A modern development platform needs excellent documentation. But documentation infrastructure shouldn't be expensive or complex to maintain. In this post, we'll deploy a static documentation site to Azure Storage with Azure Front Door for global distribution - all managed through Terraform Cloud and automated with Azure DevOps pipelines.

This approach gives us:
- **Low Cost**: Azure Storage static websites cost pennies per month
- **Global Performance**: Front Door CDN distributes content worldwide
- **Automated Deployment**: Azure DevOps pipelines rebuild and deploy on every commit
- **Infrastructure as Code**: Terraform Cloud manages all Azure resources
- **Developer-Friendly**: TechDocs generates beautiful documentation from Markdown

The code for this post is available in the [blog-platform-aspire repository](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-static-site).

## Why Static Site Hosting on Azure Storage?

Traditional web hosting requires web servers, compute resources, and ongoing maintenance. Static site hosting on Azure Storage eliminates all of that:

**Cost Comparison:**
- App Service Basic (B1): ~$13/month
- Azure Storage static website: ~$0.50/month
- Savings: 96% reduction

**What You Get:**
- 99.9% availability SLA
- Automatic scaling
- HTTPS support (with Front Door)
- Custom domain support
- CDN integration
- No server management

**Perfect For:**
- Documentation sites
- Marketing pages
- Single-page applications
- Static blog hosting
- Design system showcases

## Architecture Overview

Our documentation deployment pipeline looks like this:

```
┌─────────────────┐
│  Git Repository │
│   (Markdown)    │
└────────┬────────┘
         │
         │ Push triggers pipeline
         │
┌────────▼────────────┐
│  Azure DevOps       │
│  Pipeline           │
│  - Build TechDocs   │
│  - Run Tests        │
│  - Deploy to Azure  │
└────────┬────────────┘
         │
         │ Upload static files
         │
┌────────▼────────────┐
│  Azure Storage      │
│  Static Website     │
│  - HTML/CSS/JS      │
│  - Images/Assets    │
└────────┬────────────┘
         │
         │ Content distribution
         │
┌────────▼────────────┐
│  Azure Front Door   │
│  - Global CDN       │
│  - SSL/TLS          │
│  - Custom Domain    │
│  - Caching          │
└─────────────────────┘
```

## What is TechDocs?

[TechDocs](https://backstage.io/docs/features/techdocs/techdocs-overview) is the documentation system built into Backstage. It converts Markdown files into beautiful, searchable documentation sites with:

- **Markdown-Based**: Write docs in plain Markdown with your code
- **Built-in Search**: Full-text search across all documentation
- **Component-Aware**: Documentation tied to specific services/components
- **Beautiful UI**: Modern, responsive design out of the box
- **Version Control**: Docs live with code, versioned together

Even if you're not using Backstage, TechDocs can generate standalone documentation sites.

## Setting Up Terraform Cloud

Before we create Azure resources, we need to set up Terraform Cloud to manage our infrastructure state and execute our deployments.

### Creating the Workspace in Terraform Cloud

1. **Log into Terraform Cloud** at https://app.terraform.io
2. **Create a new workspace**:
   - Click "New workspace"
   - Choose "CLI-driven workflow"
   - Name it `documentation`
   - Add a description: "Platform documentation infrastructure"

3. **Configure workspace settings**:
   - Go to Settings → General
   - Set Execution Mode: "Remote"
   - Set Terraform Version: "1.5.0" or later
   - Enable "Auto apply" if you want automatic deployments (optional)

### Configure Workspace Variables

In the workspace, go to Variables and add the following:

**Environment Variables** (for Azure authentication):
- `ARM_CLIENT_ID`: Your Azure service principal client ID
- `ARM_CLIENT_SECRET`: Your Azure service principal secret (**mark as sensitive**)
- `ARM_SUBSCRIPTION_ID`: Your Azure subscription ID
- `ARM_TENANT_ID`: Your Azure tenant ID

**Terraform Variables**:
- `environment`: `prod` (or `dev` for development)
- `location`: `westus2`
- `project_name`: `platformdocs`
- `custom_domain`: Leave empty or set to your custom domain (e.g., `docs.yourdomain.com`)

### Getting Azure Service Principal Credentials

If you don't have a service principal yet, create one:

```bash
# Login to Azure
az login

# Create service principal
az ad sp create-for-rbac \
  --name "sp-terraform-platform-docs" \
  --role Contributor \
  --scopes /subscriptions/{subscription-id}

# Output will show:
# {
#   "appId": "...",        # Use this for ARM_CLIENT_ID
#   "password": "...",     # Use this for ARM_CLIENT_SECRET
#   "tenant": "..."        # Use this for ARM_TENANT_ID
# }

# Get your subscription ID
az account show --query id -o tsv  # Use this for ARM_SUBSCRIPTION_ID
```

## Azure Infrastructure with Terraform

Let's build the infrastructure step by step. We'll create all the Terraform files in the `documentation/infra` folder.

### Terraform Provider Configuration

Create `documentation/infra/provider.tf` to configure the AzureRM provider and Terraform Cloud backend:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.50"
    }
  }

  cloud {
    organization = "your-org-name"  # Replace with your Terraform Cloud organization
    
    workspaces {
      name = "documentation"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
  }
}
```

### Resource Group and Common Resources

Create `documentation/infra/main.tf` for the resource group and common tags:

```hcl
locals {
  common_tags = {
    environment = var.environment
    purpose     = "platform-documentation"
    managed_by  = "terraform"
    project     = var.project_name
  }
}

resource "azurerm_resource_group" "docs" {
  name     = "rg-${var.project_name}-${var.environment}"
  location = var.location

  tags = local.common_tags
}
```

### Storage Account for Static Website

Create `documentation/infra/storage.tf` for the storage account configured for static website hosting:

```hcl
resource "azurerm_storage_account" "docs" {
  name                     = "st${var.project_name}${var.environment}"
  resource_group_name      = azurerm_resource_group.docs.name
  location                 = azurerm_resource_group.docs.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"

  static_website {
    index_document     = "index.html"
    error_404_document = "404.html"
  }

  tags = local.common_tags
}

# Output the primary endpoint
output "static_website_url" {
  value       = azurerm_storage_account.docs.primary_web_endpoint
  description = "The URL of the static website"
}

output "storage_account_name" {
  value       = azurerm_storage_account.docs.name
  description = "The name of the storage account"
}
```

### Azure Front Door for Global Distribution

Create `documentation/infra/frontdoor.tf` for Azure Front Door CDN and custom domain support:

```hcl
# frontdoor.tf

resource "azurerm_cdn_frontdoor_profile" "docs" {
  name                = "fd-${var.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.docs.name
  sku_name            = "Standard_AzureFrontDoor"

  tags = local.common_tags
}

resource "azurerm_cdn_frontdoor_endpoint" "docs" {
  name                     = "ep-${var.project_name}-${var.environment}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id
  
  tags = local.common_tags
}

resource "azurerm_cdn_frontdoor_origin_group" "docs" {
  name                     = "og-${var.project_name}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id

  load_balancing {
    sample_size                 = 4
    successful_samples_required = 3
  }

  health_probe {
    path                = "/"
    request_type        = "HEAD"
    protocol            = "Https"
    interval_in_seconds = 100
  }
}

resource "azurerm_cdn_frontdoor_origin" "docs" {
  name                          = "origin-${var.project_name}"
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.docs.id

  enabled                        = true
  host_name                      = replace(replace(azurerm_storage_account.docs.primary_web_endpoint, "https://", ""), "/", "")
  http_port                      = 80
  https_port                     = 443
  origin_host_header             = replace(replace(azurerm_storage_account.docs.primary_web_endpoint, "https://", ""), "/", "")
  priority                       = 1
  weight                         = 1000
  certificate_name_check_enabled = true
}

resource "azurerm_cdn_frontdoor_route" "docs" {
  name                          = "route-${var.project_name}"
  cdn_frontdoor_endpoint_id     = azurerm_cdn_frontdoor_endpoint.docs.id
  cdn_frontdoor_origin_group_id = azurerm_cdn_frontdoor_origin_group.docs.id
  cdn_frontdoor_origin_ids      = [azurerm_cdn_frontdoor_origin.docs.id]

  supported_protocols    = ["Http", "Https"]
  patterns_to_match      = ["/*"]
  forwarding_protocol    = "HttpsOnly"
  link_to_default_domain = true
  https_redirect_enabled = true
}

output "frontdoor_endpoint" {
  value       = azurerm_cdn_frontdoor_endpoint.docs.host_name
  description = "The Front Door endpoint hostname"
}
```

### Variables and Resource Group

Create `documentation/infra/variables.tf` for the input variables:

```hcl
variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "platformdocs"
  
  validation {
    condition     = length(var.project_name) <= 15 && can(regex("^[a-z0-9]+$", var.project_name))
    error_message = "Project name must be 15 characters or less and contain only lowercase letters and numbers."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "test", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, test, staging, or prod."
  }
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "westus2"
}

variable "custom_domain" {
  description = "Custom domain for documentation site (optional)"
  type        = string
  default     = ""
}
```

### Deploying the Infrastructure

Now that all the Terraform files are created in `documentation/infra/`, let's initialize and deploy:

**First-time setup:**

```bash
# Navigate to the infrastructure folder
cd documentation/infra

# Login to Terraform Cloud
terraform login

# Initialize Terraform (downloads providers, configures backend)
terraform init
```

The `terraform init` command will:
- Download the AzureRM provider
- Configure the Terraform Cloud backend
- Link to the `documentation` workspace

**Deploy the infrastructure:**

```bash
# Format the code
terraform fmt

# Validate the configuration
terraform validate

# Plan the infrastructure changes
terraform plan

# Apply the changes (deploys to Azure)
terraform apply
```

Terraform Cloud will show you the resources being created:
- Storage Account with static website hosting
- Front Door profile and endpoint
- Origin group and origin
- Route configuration

**Cost Estimate:**
- Storage Account: ~$0.50/month (minimal traffic)
- Front Door Standard: ~$35/month (base rate)
- **Total: ~$36/month**

## Setting Up TechDocs

TechDocs requires a specific project structure. Here's a minimal setup:

### Documentation Structure

```
repo-root/
├── documentation/       # Main documentation folder
│   ├── mkdocs.yml      # TechDocs configuration
│   ├── docs/           # Markdown documentation files
│   │   ├── index.md    # Home page
│   │   ├── getting-started/
│   │   │   ├── index.md
│   │   │   └── quickstart.md
│   │   ├── architecture/
│   │   │   ├── index.md
│   │   │   └── decisions.md
│   │   └── api/
│   │       ├── index.md
│   │       └── reference.md
│   ├── site/           # Generated output (gitignored)
│   └── infra/          # Terraform infrastructure
│       ├── provider.tf
│       ├── main.tf
│       ├── variables.tf
│       ├── storage.tf
│       ├── frontdoor.tf
│       └── monitoring.tf
```

### MkDocs Configuration

Create `documentation/mkdocs.yml`:

```yaml
site_name: Platform Documentation
site_description: Modern Development Platform Documentation
site_author: Your Team

repo_url: https://github.com/yourorg/platform-docs
edit_uri: edit/main/documentation/docs/

# Markdown files are in the docs/ subfolder
docs_dir: docs

theme:
  name: material
  palette:
    primary: indigo
    accent: indigo
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.expand
    - search.suggest
    - search.highlight

nav:
  - Home: index.md
  - Getting Started:
      - Overview: getting-started/index.md
      - Quickstart: getting-started/quickstart.md
  - Architecture:
      - Overview: architecture/index.md
      - Decisions: architecture/decisions.md
  - API Reference:
      - Overview: api/index.md
      - Reference: api/reference.md

markdown_extensions:
  - admonition
  - codehilite
  - pymdownx.superfences
  - pymdownx.tabbed
  - toc:
      permalink: true

plugins:
  - search
  - techdocs-core
```

### Building TechDocs Locally

Install TechDocs CLI and test locally:

```bash
# Install TechDocs CLI
npm install -g @techdocs/cli

# Or use npx
npx @techdocs/cli

# Navigate to the documentation folder (where mkdocs.yml is located)
cd documentation

# Generate documentation (requires Docker)
techdocs-cli generate --source-dir . --output-dir ./site

# Preview locally
techdocs-cli serve --source-dir .
```

Open `http://localhost:3000` to preview your documentation.

**Troubleshooting TechDocs Generation:**

If you get a Docker error like `Docker container returned a non-zero exit code (1)`:

1. **Ensure Docker is running**: TechDocs requires Docker to be running
   ```bash
   docker ps
   ```

2. **Use the no-docker option**: Generate without Docker for local testing
   ```bash
   techdocs-cli generate --source-dir . --output-dir ./site --no-docker
   ```

3. **Install dependencies locally**: If using `--no-docker`, install Python dependencies
   ```bash
   # Install Python 3 if not already installed (macOS)
   brew install python3
   
   # Or on Ubuntu/Debian
   sudo apt-get install python3 python3-pip
   
   # Install mkdocs-techdocs-core (user-level to avoid permission issues)
   pip3 install --user mkdocs-techdocs-core
   
   # Or use a virtual environment (recommended)
   python3 -m venv venv
   source venv/bin/activate
   pip install mkdocs-techdocs-core
   ```

4. **Check mkdocs.yml**: Ensure your configuration is valid
   ```bash
   mkdocs build --strict
   ```

5. **Common mkdocs errors**:
   
   **Missing dependencies**: If you get module import errors
   ```bash
   pip install --user mkdocs-material pymdown-extensions
   ```
   
   **Invalid navigation**: Check your `nav` section in `mkdocs.yml` for typos or missing files
   ```bash
   # Test build to see specific error (run from documentation folder where mkdocs.yml is)
   cd documentation
   mkdocs build --verbose
   ```
   
   **Plugin errors**: Ensure `techdocs-core` plugin is properly installed
   ```bash
   pip show mkdocs-techdocs-core
   ```

For CI/CD pipelines, you can use the `--no-docker` flag to avoid Docker dependency:

```yaml
- script: |
    python3 -m pip install --upgrade pip
    pip3 install mkdocs-techdocs-core
    # Run from documentation folder where mkdocs.yml is located
    cd documentation
    techdocs-cli generate --source-dir . --output-dir ./site --no-docker
  displayName: 'Generate Documentation'
```

## Azure DevOps Pipeline

Now let's automate the deployment with Azure DevOps.

### Pipeline Configuration

Create `azure-pipelines.yml` in your repository:

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - documentation/docs/**
      - documentation/mkdocs.yml
      - azure-pipelines.yml

pool:
  vmImage: 'ubuntu-latest'

variables:
  - name: storageAccount
    value: 'stplatformdocsprod'
  - name: containerName
    value: '$web'

stages:
  - stage: Build
    displayName: 'Build Documentation'
    jobs:
      - job: BuildDocs
        displayName: 'Build TechDocs'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '24.x'
            displayName: 'Install Node.js'

          - script: |
              npm install -g @techdocs/cli
            displayName: 'Install TechDocs CLI'

          - script: |
              cd documentation
              techdocs-cli generate --source-dir . --output-dir ./site
            displayName: 'Generate Documentation'

          - task: PublishBuildArtifacts@1
            inputs:
              PathtoPublish: 'documentation/site'
              ArtifactName: 'documentation'
              publishLocation: 'Container'
            displayName: 'Publish Documentation Artifact'

  - stage: Deploy
    displayName: 'Deploy to Azure'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployDocs
        displayName: 'Deploy to Storage'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: DownloadBuildArtifacts@0
                  inputs:
                    buildType: 'current'
                    downloadType: 'single'
                    artifactName: 'documentation'
                    downloadPath: '$(System.ArtifactsDirectory)'
                  displayName: 'Download Documentation'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'documentation'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Remove old files
                      az storage blob delete-batch --account-name $(storageAccount) --source $(containerName) --auth-mode login
                      
                      # Upload new files
                      az storage blob upload-batch --account-name $(storageAccount) --destination $(containerName) --source $(System.ArtifactsDirectory)/documentation --overwrite true --auth-mode login
                  displayName: 'Deploy to Storage Account'

                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'documentation'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Purge Front Door cache
                      az afd endpoint purge --resource-group rg-platformdocs-prod --profile-name fd-platformdocs-prod --endpoint-name ep-platformdocs-prod --content-paths "/*"
                  displayName: 'Purge Front Door Cache'
```

### Service Connection

Create an Azure Resource Manager service connection in Azure DevOps:

1. Go to Project Settings → Service connections
2. Create new service connection → Azure Resource Manager
3. **Authentication method**: Select "Workload Identity federation (automatic)"
4. **Scope level**: Choose your subscription
5. **Resource group**: Select the resource group containing your storage account
6. **Service connection name**: `documentation`
7. Click "Save"

Azure DevOps will automatically create an App Registration in your Azure AD and configure the workload identity federation for secure, keyless authentication.

### Grant Storage Permissions

The service principal needs permission to manage blobs in the storage account:

```bash
# Get the service principal ID from the service connection
# (You can find this in Azure DevOps: Service Connection → Manage Service Principal)

# Assign Storage Blob Data Owner role
az role assignment create \
  --assignee <service-principal-object-id> \
  --role "Storage Blob Data Owner" \
  --scope /subscriptions/<subscription-id>/resourceGroups/rg-platformdocs-prod/providers/Microsoft.Storage/storageAccounts/stplatformdocsprod
```

Alternatively, assign the role through the Azure Portal:

1. Navigate to your storage account in the Azure Portal
2. Click "Access Control (IAM)" in the left menu
3. Click "+ Add" → "Add role assignment"
4. Select "Storage Blob Data Owner" role
5. Click "Next"
6. Select "User, group, or service principal"
7. Click "+ Select members"
8. Search for your service connection name (`documentation`)
9. Select it and click "Select"
10. Click "Review + assign"

## Custom Domain Configuration (Optional)

To use a custom domain like `docs.yourdomain.com`:

### Add Custom Domain to Front Door

```hcl
# Add to frontdoor.tf

resource "azurerm_cdn_frontdoor_custom_domain" "docs" {
  count = var.custom_domain != "" ? 1 : 0

  name                     = "custom-domain-${var.project_name}"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.docs.id
  host_name                = var.custom_domain

  tls {
    certificate_type    = "ManagedCertificate"
    minimum_tls_version = "TLS12"
  }
}

resource "azurerm_cdn_frontdoor_custom_domain_association" "docs" {
  count = var.custom_domain != "" ? 1 : 0

  cdn_frontdoor_custom_domain_id = azurerm_cdn_frontdoor_custom_domain.docs[0].id
  cdn_frontdoor_route_ids        = [azurerm_cdn_frontdoor_route.docs.id]
}
```

### DNS Configuration

Add a CNAME record to your DNS:

```
docs.yourdomain.com  CNAME  ep-platformdocs-prod-xxxxx.azurefd.net
```

Front Door will automatically provision a managed SSL certificate.

## Monitoring and Analytics

Add monitoring to track documentation usage:

### Application Insights Integration

Create `documentation/infra/monitoring.tf` for Application Insights:

```hcl

resource "azurerm_application_insights" "docs" {
  name                = "ai-${var.project_name}-${var.environment}"
  resource_group_name = azurerm_resource_group.docs.name
  location            = azurerm_resource_group.docs.location
  application_type    = "web"

  tags = local.common_tags
}

output "instrumentation_key" {
  value       = azurerm_application_insights.docs.instrumentation_key
  sensitive   = true
  description = "Application Insights instrumentation key"
}

output "connection_string" {
  value       = azurerm_application_insights.docs.connection_string
  sensitive   = true
  description = "Application Insights connection string"
}
```

Add the Application Insights snippet to your documentation template.

## Testing the Deployment

After your pipeline runs, test the deployment:

```bash
# Test the storage endpoint directly
curl https://stplatformdocsprod.z5.web.core.windows.net/

# Test through Front Door
curl https://ep-platformdocs-prod-xxxxx.azurefd.net/

# Test custom domain (if configured)
curl https://docs.yourdomain.com/
```

## Best Practices

### 1. Cache Control Headers

Configure cache headers for better performance:

```bash
# Set cache headers on upload
az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT \
  --destination '$web' \
  --source ./site \
  --content-cache-control "public, max-age=3600"
```

### 2. Compression

Enable compression in Front Door for better performance:

```hcl
resource "azurerm_cdn_frontdoor_rule" "compression" {
  name                      = "compression"
  cdn_frontdoor_route_id    = azurerm_cdn_frontdoor_route.docs.id
  order                     = 1
  behavior_on_match         = "Continue"

  actions {
    response_header_action {
      header_action = "Append"
      header_name   = "Content-Encoding"
      value         = "gzip"
    }
  }

  conditions {
    request_header_condition {
      header_name  = "Accept-Encoding"
      operator     = "Contains"
      match_values = ["gzip"]
    }
  }
}
```

### 3. Security Headers

Add security headers to your documentation:

```hcl
resource "azurerm_cdn_frontdoor_rule" "security_headers" {
  name                      = "security-headers"
  cdn_frontdoor_route_id    = azurerm_cdn_frontdoor_route.docs.id
  order                     = 2
  behavior_on_match         = "Continue"

  actions {
    response_header_action {
      header_action = "Append"
      header_name   = "X-Content-Type-Options"
      value         = "nosniff"
    }

    response_header_action {
      header_action = "Append"
      header_name   = "X-Frame-Options"
      value         = "DENY"
    }

    response_header_action {
      header_action = "Append"
      header_name   = "Strict-Transport-Security"
      value         = "max-age=31536000; includeSubDomains"
    }
  }
}
```

## Troubleshooting

### Documentation Not Updating

If your documentation doesn't update after deployment:

1. **Check the pipeline**: Verify the build and deploy stages succeeded
2. **Purge the cache**: Front Door caches content aggressively
   ```bash
   az afd endpoint purge \
     --resource-group rg-platformdocs-prod \
     --profile-name fd-platformdocs-prod \
     --endpoint-name ep-platformdocs-prod \
     --content-paths "/*"
   ```
3. **Verify upload**: Check the storage account to ensure files were uploaded
   ```bash
   az storage blob list \
     --account-name stplatformdocsprod \
     --container-name '$web' \
     --output table
   ```

### 404 Errors

If you get 404 errors:

1. **Check index document**: Ensure `index.html` exists in the root
2. **Verify static website**: Confirm static website hosting is enabled
3. **Check routing**: Verify Front Door route configuration

### Custom Domain Not Working

If your custom domain doesn't work:

1. **Verify DNS**: Check CNAME record propagation
   ```bash
   nslookup docs.yourdomain.com
   ```
2. **Check certificate**: Ensure the managed certificate is provisioned (can take 15-30 minutes)
3. **Review association**: Verify the custom domain association in Front Door

## Cost Optimization

Our documentation setup is already cost-effective, but you can optimize further:

### 1. Use Storage Account LRS

We're already using Locally Redundant Storage (LRS) which is the cheapest option.

### 2. Monitor Front Door Usage

Front Door Standard costs ~$35/month base rate plus:
- $0.01 per GB of data transfer
- $0.20 per million requests

For a documentation site with moderate traffic:
- 10 GB transfer/month: $0.10
- 100K requests/month: $0.02

### 3. Consider Front Door Classic

For very low traffic sites, Front Door Classic might be cheaper:
- No base rate
- Pay only for usage
- Less features, but sufficient for documentation

### Monthly Cost Breakdown

**Production Setup:**
- Storage Account: $0.50
- Front Door Standard: $35.00
- Data transfer (10 GB): $0.10
- **Total: ~$36/month**

**Development Setup:**
- Storage Account only: $0.50
- Use direct storage endpoint
- **Total: ~$0.50/month**

## Cleaning Up Resources

To delete all resources and avoid charges:

```bash
# Through Terraform Cloud
terraform destroy

# Confirm the destruction
# This will remove:
# - Front Door profile and endpoints
# - Storage account and all content
# - Resource group
```

## Coming up in the Series

In the next posts, we'll continue building our platform infrastructure:

- **November 4**: Generating type-safe API clients with Kiota
- **November 11**: Creating reusable Azure DevOps pipeline templates
- **November 18**: Building custom .NET project templates for standardization
- **November 25**: Service virtualization with WireMock for development
- **December 2**: Setting up Azure subscription structure with Terraform
- **December 9**: Designing Azure environment architecture (dev/test/prod)
- **December 16**: Building shared services infrastructure
- **December 23**: Configuring RBAC and permissions with Terraform
- **December 30**: Exploring Pulumi as an alternative to Terraform

## Summary

We've built a complete documentation deployment pipeline that:

✅ Hosts static documentation on Azure Storage for minimal cost
✅ Distributes globally with Azure Front Door CDN
✅ Generates beautiful docs with TechDocs
✅ Automates deployment with Azure DevOps pipelines
✅ Manages infrastructure with Terraform Cloud
✅ Supports custom domains with managed SSL certificates
✅ Costs only ~$36/month for production (or $0.50 for dev)

This infrastructure gives us enterprise-grade documentation hosting at a fraction of the cost of traditional web hosting, with the reliability and performance of Azure's global infrastructure.

The pattern we've built here can be used for any static site - documentation, marketing pages, SPAs, or blogs. The combination of Azure Storage, Front Door, and automated CI/CD creates a robust, scalable, and cost-effective hosting solution.
