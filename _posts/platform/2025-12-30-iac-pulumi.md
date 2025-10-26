---
title: "Building a Modern Development Platform: Pulumi as an Alternative to Terraform for Azure IaC 🔄"
date: 2025-12-30T06:00:00-07:00
draft: false
categories: ["platform","pulumi","iac","azure","infrastructure","terraform"]
description: "Exploring Pulumi as an alternative to Terraform for Azure infrastructure as code - leveraging familiar programming languages for type safety and better developer experience"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction 🚀

In our [Terraform post](https://brianpsheridan.com/platform/terraform/iac/azure/infrastructure/terraform-cloud/2025/10/21/tools-terraform.html), we explored using Terraform and Terraform Cloud for Infrastructure as Code. But what if you could write infrastructure code in the same programming languages you use for your applications? That's where **Pulumi** comes in.

Pulumi offers a compelling alternative to Terraform by letting you define infrastructure using familiar languages like TypeScript, Python, C#, and Go. This brings several advantages:

- **Type safety** - Catch errors at compile time, not runtime
- **Better IDE support** - IntelliSense, refactoring, debugging
- **Familiar syntax** - Use the language your team already knows
- **Standard tooling** - npm, pip, NuGet, go modules
- **Programmatic flexibility** - Loops, conditionals, functions without HCL limitations

This post explores when Pulumi makes sense, how it compares to Terraform, and walks through implementing the same three-tier Azure architecture we built with Terraform.

**📦 Code Repository**: The Pulumi version of our infrastructure code is available at [blog-platform-aspire/aspire-tools-pulumi](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-pulumi).

## Pulumi vs Terraform: When to Choose What? 🤔

### Choose Terraform When:
- ✅ You need the **largest ecosystem** of providers and modules
- ✅ Your team is **already familiar with HCL** and Terraform workflows
- ✅ You want **declarative** infrastructure definitions
- ✅ You need **HashiCorp ecosystem integration** (Vault, Consul, Nomad)
- ✅ You have **existing Terraform state** and modules to maintain
- ✅ You prefer **provider-agnostic DSL** (HCL) over general-purpose languages

### Choose Pulumi When:
- ✅ You want **strong type safety** and compile-time validation
- ✅ Your team prefers **general-purpose languages** (TypeScript, Python, C#, Go)
- ✅ You need **complex logic** that's awkward in HCL (dynamic resources, advanced loops)
- ✅ You want **better IDE support** (IntelliSense, refactoring, debugging)
- ✅ You value **testing with standard frameworks** (Jest, pytest, xUnit)
- ✅ You want infrastructure code that **feels like application code**
- ✅ You need **policy as code** with strong language support

### Key Differences

| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| **Language** | HCL (declarative DSL) | TypeScript, Python, C#, Go, Java, YAML |
| **Type Safety** | Limited | Strong (compile-time) |
| **IDE Support** | Basic | Full (IntelliSense, refactoring) |
| **Testing** | Terratest (external) | Native frameworks (Jest, pytest, xUnit) |
| **State Management** | Terraform Cloud or backends | Pulumi Service or self-managed |
| **Module Sharing** | Terraform Registry | npm, pip, NuGet, Go modules |
| **Learning Curve** | Learn HCL | Use existing language skills |
| **Ecosystem Size** | Largest (3000+ providers) | Growing (100+ providers) |
| **Policy as Code** | Sentinel (proprietary) | CrossGuard (open, language-native) |
| **Cost** | Free tier: 500 resources/month | Free tier: Individual use, unlimited resources |

### Real-World Considerations

**Terraform Excels At:**
- Multi-cloud scenarios with complex provider dependencies
- Organizations with heavy investment in HashiCorp ecosystem
- Teams that prefer infrastructure-specific tooling
- Scenarios where provider maturity is critical

**Pulumi Excels At:**
- Developer-heavy teams already proficient in TypeScript/Python/C#
- Projects requiring complex conditional logic or dynamic resource creation
- Organizations wanting infrastructure code that aligns with application development practices
- Teams prioritizing type safety and IDE-driven development

**The Hybrid Approach:**

You don't have to choose one exclusively. Many teams use:
- **Terraform** for foundational infrastructure (networking, identity, shared services)
- **Pulumi** for application-specific resources (where developer experience matters most)

This leverages Terraform's ecosystem maturity for stable, slowly-changing infrastructure while using Pulumi's developer-friendly approach for frequently-updated application resources.

## Pulumi Setup 🛠️

### Prerequisites

Before getting started, ensure you have:
- **Azure CLI** installed and authenticated (`az login`)
- **Node.js 18+** (for TypeScript) or **Python 3.8+** / **.NET 6+** (depending on your language choice)
- **Azure subscription** with appropriate permissions

### Installing Pulumi CLI

**macOS (using Homebrew):**
```bash
brew install pulumi/tap/pulumi

# Verify installation
pulumi version
```

**Windows (using Chocolatey):**
```bash
choco install pulumi

# Verify installation
pulumi version
```

**Linux (using install script):**
```bash
curl -fsSL https://get.pulumi.com | sh

# Verify installation
pulumi version
```

### Pulumi Service vs Self-Managed Backend

Pulumi offers two state management options:

**1. Pulumi Service (Recommended for Getting Started)**
- Free for individual use with unlimited resources
- Built-in secrets management
- Web-based state inspection and history
- Team collaboration features in paid tiers

**2. Self-Managed Backend**
- Azure Blob Storage
- AWS S3
- Local filesystem
- You manage encryption and access control

For this tutorial, we'll use the **Pulumi Service** (free tier), but I'll show how to migrate to Azure Blob Storage later.

### Authenticating with Pulumi Service

```bash
# Login to Pulumi Service (opens browser)
pulumi login

# Alternative: Login with access token
pulumi login --token <your-token>

# Verify login
pulumi whoami
```

## Azure Authentication with Pulumi 🔐

Pulumi supports multiple Azure authentication methods. We'll use **Service Principal with Client Secret** for simplicity, but in production, you should use **Managed Identity** or **OIDC** (similar to our Terraform setup).

### Creating Azure Service Principal

```bash
# Login to Azure
az login

# Set your subscription
SUBSCRIPTION_ID="your-subscription-id"
az account set --subscription $SUBSCRIPTION_ID

# Create service principal
SP_NAME="pulumi-platform-sp"

az ad sp create-for-rbac \
  --name $SP_NAME \
  --role Contributor \
  --scopes "/subscriptions/$SUBSCRIPTION_ID" \
  --output json > sp-output.json

# Capture the values
CLIENT_ID=$(cat sp-output.json | jq -r '.appId')
CLIENT_SECRET=$(cat sp-output.json | jq -r '.password')
TENANT_ID=$(cat sp-output.json | jq -r '.tenant')

echo "Client ID: $CLIENT_ID"
echo "Tenant ID: $TENANT_ID"
echo "Subscription ID: $SUBSCRIPTION_ID"

# Store client secret securely (we'll use it with Pulumi config)
# DO NOT commit sp-output.json to git
```

### Configuring Pulumi with Azure Credentials

Pulumi can use Azure credentials in several ways:

**Option 1: Environment Variables (Quick Testing)**
```bash
export ARM_CLIENT_ID="$CLIENT_ID"
export ARM_CLIENT_SECRET="$CLIENT_SECRET"
export ARM_TENANT_ID="$TENANT_ID"
export ARM_SUBSCRIPTION_ID="$SUBSCRIPTION_ID"
```

**Option 2: Pulumi Config (Recommended)**

We'll set these per-stack using Pulumi's encrypted config:

```bash
# Set Azure config (encrypted)
pulumi config set azure-native:clientId $CLIENT_ID
pulumi config set azure-native:clientSecret $CLIENT_SECRET --secret
pulumi config set azure-native:tenantId $TENANT_ID
pulumi config set azure-native:subscriptionId $SUBSCRIPTION_ID
```

**Option 3: Azure CLI Authentication (Local Development)**

If you're already authenticated with Azure CLI, Pulumi can use those credentials automatically - no additional configuration needed!

```bash
# Just ensure you're logged in
az login
az account set --subscription $SUBSCRIPTION_ID
```

## Infrastructure Code Organization 📁

We'll replicate the same three-tier architecture from our Terraform post, but using TypeScript for type safety and better developer experience.

### Project Structure

```
pulumi-infra/
├── shared-services/           # Cross-environment shared resources
│   ├── index.ts              # Main program
│   ├── Pulumi.yaml           # Project definition
│   ├── Pulumi.dev.yaml       # Stack config (dev)
│   ├── package.json          # npm dependencies
│   └── tsconfig.json         # TypeScript config
│
├── environment/              # Environment-level infrastructure
│   ├── index.ts
│   ├── Pulumi.yaml
│   ├── Pulumi.test.yaml      # Stack config (test)
│   ├── package.json
│   └── tsconfig.json
│
└── app/                      # Application-specific resources
    ├── index.ts
    ├── Pulumi.yaml
    ├── Pulumi.test.yaml
    ├── package.json
    └── tsconfig.json
```

**Why TypeScript?**
- Strong type safety with Azure resource types
- Excellent IDE support (IntelliSense, auto-completion)
- Familiar to many developers
- Easy async/await for dependencies

You could also use Python, C#, or Go - the concepts remain the same.

## Layer 1: Shared Services Infrastructure 🏗️

Let's create the shared services stack with Azure Container Registry.

### Initialize the Project

```bash
mkdir -p pulumi-infra/shared-services
cd pulumi-infra/shared-services

# Create new Pulumi project
pulumi new azure-typescript --name shared-services --description "Shared services infrastructure"

# This creates:
# - Pulumi.yaml (project definition)
# - index.ts (infrastructure code)
# - package.json (dependencies)
# - tsconfig.json (TypeScript config)
```

### Install Dependencies

```bash
# Azure Native provider (latest Azure ARM API)
npm install @pulumi/azure-native

# Random provider for unique names
npm install @pulumi/random
```

### Shared Services Code

**`index.ts`**
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as resources from "@pulumi/azure-native/resources";
import * as containerregistry from "@pulumi/azure-native/containerregistry";
import * as random from "@pulumi/random";

// Configuration
const config = new pulumi.Config();
const location = config.get("location") || "westus2";
const tags = {
    ManagedBy: "Pulumi",
    Project: "Platform",
    Layer: "Shared-Services",
};

// Create resource group
const resourceGroup = new resources.ResourceGroup("rg-shared-services", {
    resourceGroupName: "rg-shared-services",
    location: location,
    tags: tags,
});

// Generate random suffix for ACR (must be globally unique)
const suffix = new random.RandomString("acr-suffix", {
    length: 6,
    special: false,
    upper: false,
});

// Create Azure Container Registry
const acr = new containerregistry.Registry("acr-shared", {
    registryName: pulumi.interpolate`acrshared${suffix.result}`,
    resourceGroupName: resourceGroup.name,
    location: resourceGroup.location,
    sku: {
        name: "Basic",
    },
    adminUserEnabled: false,
    tags: tags,
});

// Export outputs for other stacks to consume
export const resourceGroupName = resourceGroup.name;
export const acrId = acr.id;
export const acrLoginServer = acr.loginServer;
export const acrName = acr.name;
```

### Configure and Deploy

```bash
# Configure Azure subscription and location
pulumi config set azure-native:location westus2

# If using service principal (Option 2 from earlier)
pulumi config set azure-native:clientId $CLIENT_ID
pulumi config set azure-native:clientSecret $CLIENT_SECRET --secret
pulumi config set azure-native:tenantId $TENANT_ID
pulumi config set azure-native:subscriptionId $SUBSCRIPTION_ID

# Preview changes
pulumi preview

# Deploy
pulumi up

# View outputs
pulumi stack output acrLoginServer
```

**Key Differences from Terraform:**
- No `terraform.tf` or `provider` blocks - defined in `Pulumi.yaml`
- Resources are TypeScript objects with IntelliSense
- `pulumi.interpolate` for string templating (like `${...}` in Terraform)
- Outputs are exported TypeScript variables
- Async/await for handling dependencies (though Pulumi handles this automatically)

## Layer 2: Environment Infrastructure 🌐

Now let's create the environment-level stack with App Service Plan.

### Initialize Environment Project

```bash
cd ../
mkdir environment
cd environment

pulumi new azure-typescript --name environment --stack test
npm install @pulumi/azure-native
```

### Environment Code

**`index.ts`**
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as resources from "@pulumi/azure-native/resources";
import * as web from "@pulumi/azure-native/web";

// Configuration
const config = new pulumi.Config();
const environment = config.require("environment"); // "test", "dev", "staging", "prod"
const location = config.get("location") || "westus2";

// Reference shared services stack
const sharedStack = new pulumi.StackReference("shared-services-stack", {
    name: "organization/shared-services/dev", // Replace with your org/project/stack
});

const tags = {
    ManagedBy: "Pulumi",
    Project: "Platform",
    Environment: environment,
};

// Create resource group
const resourceGroup = new resources.ResourceGroup(`rg-${environment}`, {
    resourceGroupName: `rg-${environment}`,
    location: location,
    tags: tags,
});

// Create App Service Plan
const appServicePlan = new web.AppServicePlan(`asp-${environment}`, {
    name: `asp-${environment}`,
    resourceGroupName: resourceGroup.name,
    location: resourceGroup.location,
    kind: "Linux",
    reserved: true, // Required for Linux
    sku: {
        name: environment === "prod" ? "P1v3" : "P0v3",
        tier: environment === "prod" ? "PremiumV3" : "PremiumV3",
    },
    tags: tags,
});

// Export outputs
export const appServicePlanId = appServicePlan.id;
export const appServicePlanName = appServicePlan.name;

// Pass through shared services outputs
export const acrLoginServer = sharedStack.getOutput("acrLoginServer");
```

### Configure Stack

**`Pulumi.test.yaml`**
```yaml
config:
  environment:environment: "test"
  azure-native:location: "westus2"
```

### Deploy

```bash
# Set environment
pulumi config set environment test

# Deploy
pulumi up

# View outputs
pulumi stack output appServicePlanId
```

**Key Differences:**
- `StackReference` replaces Terraform's `terraform_remote_state`
- Stack references are strongly typed
- Config is YAML-based per stack
- No need for separate `variables.tf` - use `pulumi.Config()`

## Layer 3: Application Infrastructure 🚀

Finally, let's create the application stack with Web App.

### Initialize App Project

```bash
cd ../
mkdir app
cd app

pulumi new azure-typescript --name app --stack test
npm install @pulumi/azure-native
```

### Application Code

**`index.ts`**
```typescript
import * as pulumi from "@pulumi/pulumi";
import * as resources from "@pulumi/azure-native/resources";
import * as web from "@pulumi/azure-native/web";
import * as authorization from "@pulumi/azure-native/authorization";
import * as random from "@pulumi/random";

// Configuration
const config = new pulumi.Config();
const environment = config.require("environment");
const appName = config.get("appName") || "myapp";
const location = config.get("location") || "westus2";

// Reference stacks
const sharedStack = new pulumi.StackReference("shared-services", {
    name: "organization/shared-services/dev",
});

const envStack = new pulumi.StackReference("environment", {
    name: `organization/environment/${environment}`,
});

const tags = {
    ManagedBy: "Pulumi",
    Project: "Platform",
    Environment: environment,
    Application: appName,
};

// Create resource group
const resourceGroup = new resources.ResourceGroup(`rg-${environment}-${appName}`, {
    resourceGroupName: `rg-${environment}-${appName}`,
    location: location,
    tags: tags,
});

// Generate random suffix for unique web app name
const suffix = new random.RandomString("webapp-suffix", {
    length: 6,
    special: false,
    upper: false,
});

// Get values from stack references
const appServicePlanId = envStack.getOutput("appServicePlanId");
const acrLoginServer = sharedStack.getOutput("acrLoginServer");
const acrName = sharedStack.getOutput("acrName");

// Create Web App
const webApp = new web.WebApp(`app-${environment}-${appName}`, {
    name: pulumi.interpolate`app-${environment}-${appName}-${suffix.result}`,
    resourceGroupName: resourceGroup.name,
    location: resourceGroup.location,
    serverFarmId: appServicePlanId,
    kind: "app,linux,container",
    identity: {
        type: "SystemAssigned",
    },
    siteConfig: {
        linuxFxVersion: pulumi.interpolate`DOCKER|${acrLoginServer}/${appName}:latest`,
        alwaysOn: environment === "prod",
        appSettings: [
            {
                name: "DOCKER_REGISTRY_SERVER_URL",
                value: pulumi.interpolate`https://${acrLoginServer}`,
            },
            {
                name: "WEBSITES_ENABLE_APP_SERVICE_STORAGE",
                value: "false",
            },
        ],
    },
    tags: tags,
});

// Grant Web App access to ACR
const acrId = sharedStack.getOutput("acrId");
const acrPullRole = authorization.getRoleDefinition({
    roleDefinitionId: "7f951dda-4ed3-4680-a7ca-43fe172d538d", // AcrPull role
    scope: pulumi.interpolate`/subscriptions/${config.require("azure-native:subscriptionId")}`,
});

const roleAssignment = new authorization.RoleAssignment("acr-pull-assignment", {
    principalId: webApp.identity.apply(id => id!.principalId!),
    principalType: "ServicePrincipal",
    roleDefinitionId: acrPullRole.then(role => role.id),
    scope: acrId,
});

// Export outputs
export const webAppName = webApp.name;
export const webAppUrl = pulumi.interpolate`https://${webApp.defaultHostName}`;
export const webAppDefaultHostname = webApp.defaultHostName;
```

### Configure and Deploy

```bash
# Set config
pulumi config set environment test
pulumi config set appName myapp

# Deploy
pulumi up

# Get the URL
pulumi stack output webAppUrl
```

**Key Advantages:**
- Type-safe access to Azure resource properties
- IntelliSense for all Azure resources
- Compile-time validation (catch errors before deployment)
- Familiar TypeScript syntax (no new DSL to learn)
- Easy async handling with stack references

## Cross-Stack References 🔗

One of Pulumi's strengths is strongly-typed stack references.

### Exporting Outputs

In your source stack:
```typescript
export const myValue = someResource.someProperty;
```

### Consuming Outputs

In your dependent stack:
```typescript
const otherStack = new pulumi.StackReference("other-stack", {
    name: "organization/project/stack",
});

const value = otherStack.getOutput("myValue");
```

**Type Safety:**
```typescript
// Pulumi knows the type!
const acrLoginServer = sharedStack.getOutput("acrLoginServer"); // Output<string>

// Use it in resource definitions
const webApp = new web.WebApp("my-app", {
    // ...
    siteConfig: {
        linuxFxVersion: pulumi.interpolate`DOCKER|${acrLoginServer}/myapp:latest`,
    },
});
```

## Testing Infrastructure Code 🧪

One of Pulumi's best features: test your infrastructure with standard testing frameworks.

### Unit Testing with Jest

**Install test dependencies:**
```bash
npm install --save-dev @types/jest @types/node jest ts-jest
```

**`__tests__/index.test.ts`**
```typescript
import * as pulumi from "@pulumi/pulumi";

// Mock Pulumi runtime
pulumi.runtime.setMocks({
    newResource: function(args: pulumi.runtime.MockResourceArgs): {id: string, state: any} {
        return {
            id: args.inputs.name + "_id",
            state: args.inputs,
        };
    },
    call: function(args: pulumi.runtime.MockCallArgs) {
        return args.inputs;
    },
});

describe("Infrastructure", () => {
    let infra: typeof import("../index");

    beforeAll(async () => {
        infra = await import("../index");
    });

    test("Resource group name should match pattern", (done) => {
        pulumi.all([infra.resourceGroupName]).apply(([rgName]) => {
            expect(rgName).toBe("rg-shared-services");
            done();
        });
    });

    test("ACR name should have correct prefix", (done) => {
        pulumi.all([infra.acrName]).apply(([name]) => {
            expect(name).toMatch(/^acrshared/);
            done();
        });
    });
});
```

**Run tests:**
```bash
npm test
```

### Integration Testing

For more comprehensive testing, Pulumi supports integration tests:

**`integration/deployment.test.ts`**
```typescript
import * as pulumi from "@pulumi/pulumi";
import { LocalWorkspace } from "@pulumi/pulumi/automation";

test("Deploy and validate infrastructure", async () => {
    const stackName = "test-integration";
    const stack = await LocalWorkspace.createOrSelectStack({
        stackName,
        projectName: "shared-services",
        program: async () => {
            // Your infrastructure code
        },
    });

    await stack.up();
    
    const outputs = await stack.outputs();
    expect(outputs.acrLoginServer).toBeDefined();
    
    await stack.destroy();
}, 300000); // 5 minute timeout
```

## Pulumi vs Terraform: Developer Experience 👨‍💻

### Type Safety Example

**Terraform (HCL):**
```hcl
resource "azurerm_linux_web_app" "main" {
  name                = "app-${var.environment}-${var.app_name}"
  resource_group_name = azurerm_resource_group.app.name
  location            = azurerm_resource_group.app.location
  service_plan_id     = data.terraform_remote_state.environment.outputs.app_service_plan_id
  
  site_config {
    # Typo here won't be caught until runtime
    alway_on = true
  }
}
```

**Pulumi (TypeScript):**
```typescript
const webApp = new web.WebApp("app-main", {
    name: `app-${environment}-${appName}`,
    resourceGroupName: resourceGroup.name,
    location: resourceGroup.location,
    serverFarmId: appServicePlanId,
    
    siteConfig: {
        // TypeScript catches typo at compile time!
        // alwayOn: true, ✅ Correct
        // alwaysOn: true, ❌ Compile error
        alwaysOn: true,
    },
});
```

### IDE Support

**Terraform:**
- Basic syntax highlighting
- Limited auto-completion
- No refactoring support
- Errors shown at `terraform plan` time

**Pulumi:**
- Full IntelliSense
- Auto-completion for all properties
- Safe refactoring (rename, move, etc.)
- Errors shown in IDE immediately
- Jump to definition for resources
- Inline documentation

### Complex Logic

**Terraform (HCL):**
```hcl
locals {
  # Complex logic requires creative HCL usage
  web_apps = [for i in range(3) : {
    name        = "app-${i}"
    environment = i == 0 ? "dev" : "prod"
  }]
}

resource "azurerm_linux_web_app" "apps" {
  for_each = { for app in local.web_apps : app.name => app }
  
  name                = each.value.name
  # ... rest of config
}
```

**Pulumi (TypeScript):**
```typescript
// Use familiar programming constructs
const webApps = ["dev", "staging", "prod"].map((env, i) => {
    return new web.WebApp(`app-${env}`, {
        name: `app-${env}-myapp`,
        resourceGroupName: resourceGroups[env].name,
        // ... rest of config
        siteConfig: {
            // Conditional logic is natural
            alwaysOn: env === "prod",
            appSettings: [
                ...commonSettings,
                ...(env === "prod" ? productionSettings : []),
            ],
        },
    });
});
```

## State Management 💾

### Pulumi Service (Default)

Free tier includes:
- Unlimited resources
- Individual use
- State encryption
- Web-based inspection
- 100 updates/month

### Self-Managed Backend (Azure Blob Storage)

**Switch to Azure Blob Storage:**

```bash
# Create storage account for Pulumi state
az storage account create \
    --name pulumistate$RANDOM \
    --resource-group rg-pulumi-backend \
    --location westus2 \
    --sku Standard_LRS

# Create container
az storage container create \
    --name pulumi-state \
    --account-name pulumistate12345

# Login to Azure backend
pulumi login azblob://pulumi-state?storage_account=pulumistate12345

# Or set in environment
export PULUMI_BACKEND_URL="azblob://pulumi-state?storage_account=pulumistate12345"
```

**State encryption:**
```bash
# Set passphrase for state encryption
export PULUMI_CONFIG_PASSPHRASE="your-secure-passphrase"

# Or use Azure Key Vault
pulumi stack init --secrets-provider="azurekeyvault://myvault.vault.azure.net/keys/my-key"
```

## Cleaning Up Resources 🧹

Destroy in reverse order (app → environment → shared-services):

```bash
# Step 1: Destroy application
cd pulumi-infra/app
pulumi destroy -y

# Step 2: Destroy environment
cd ../environment
pulumi destroy -y

# Step 3: Destroy shared services
cd ../shared-services
pulumi destroy -y
```

**Remove stacks:**
```bash
# After destroying resources
pulumi stack rm test --yes
```

## Cost Comparison: Pulumi vs Terraform Cloud 💰

### Terraform Cloud
- **Free**: 500 resources/month
- **Beyond 500**: ~$0.00014 per resource per hour (~$100/month for 1000 resources)
- **Team Plan**: $20/user/month (includes VCS integration, policies)

### Pulumi Service
- **Individual (Free)**: Unlimited resources, single user
- **Team**: $50/user/month (team collaboration, RBAC)
- **Enterprise**: Custom pricing (SSO, audit logs, self-hosting)

### Self-Managed State (Both)
- **Azure Blob Storage**: ~$0.02/GB/month (negligible for most projects)
- No per-resource costs
- You manage encryption and access control

**Winner**: Pulumi's free tier is more generous for individual developers and small teams.

## Migration from Terraform to Pulumi 🔄

Already have Terraform? Pulumi can help:

### Import Existing Infrastructure

```bash
# Pulumi can import resources managed by Terraform
pulumi import azure-native:web:WebApp myWebApp /subscriptions/.../resourceGroups/.../providers/Microsoft.Web/sites/myapp
```

### Convert Terraform to Pulumi

```bash
# Install tf2pulumi converter
npm install -g @pulumi/tf2pulumi

# Convert Terraform HCL to Pulumi TypeScript
tf2pulumi --language typescript --out ./pulumi-code ./terraform-code
```

**Note**: Conversion is not 100% automated. You'll need to:
- Review generated code
- Adjust for Pulumi idioms
- Update stack references
- Test thoroughly

### Gradual Migration Strategy

1. **Start fresh projects with Pulumi**
2. **Import critical resources** from Terraform
3. **Migrate stable infrastructure** (networking, identity)
4. **Keep Terraform** for resources with immature Pulumi providers
5. **Use both tools** with clear boundaries

## Best Practices 🎯

### Project Organization
- **Separate stacks** for different lifecycles (shared, environment, app)
- **Use stack references** instead of monolithic stacks
- **One stack per environment** for isolated state

### Configuration Management
- **Use Pulumi config** for environment-specific values
- **Mark secrets** with `--secret` flag
- **Store config in version control** (encrypted values are safe)

### Code Quality
- **Write unit tests** for infrastructure logic
- **Use TypeScript strict mode** for maximum type safety
- **Follow language conventions** (Pulumi feels like app code)
- **Create reusable components** (classes, functions, modules)

### Security
- **Use service principal** with least privilege
- **Rotate credentials** regularly
- **Enable audit logging** in Pulumi Service
- **Use Azure Key Vault** for secrets in production

### CI/CD Integration
- **Pulumi GitHub Actions** for automated deployments
- **Azure DevOps integration** with Pulumi tasks
- **Preview on PRs** to catch issues early
- **Automatic state updates** without manual intervention

## Next Steps 🚀

We've explored Pulumi as a powerful alternative to Terraform:
- ✅ Strong type safety with familiar languages
- ✅ Better IDE support and developer experience
- ✅ Standard testing frameworks
- ✅ Flexible programming constructs
- ✅ Same three-tier architecture pattern

**Coming up in the series:**
- **Pulumi Automation API** - Programmatically manage infrastructure deployments
- **Policy as Code with CrossGuard** - Enforce compliance using TypeScript/Python
- **Component Resources** - Building reusable infrastructure abstractions
- **Pulumi + Azure DevOps** - Full CI/CD pipeline for infrastructure

## When Should You Switch? 🤔

**Stick with Terraform if:**
- Your team is happy with HCL and invested in Terraform
- You need providers not yet available in Pulumi
- You have extensive Terraform modules to maintain

**Try Pulumi if:**
- You want infrastructure code that feels like application code
- Your team prefers TypeScript/Python/C# over HCL
- You need complex conditional logic or dynamic resources
- You value strong type safety and IDE support
- You want to test infrastructure with familiar frameworks

**Both are excellent choices** - it comes down to team preferences, existing investments, and specific use cases. Many organizations successfully use both tools where each excels.

## Resources 📚

- [Pulumi Documentation](https://www.pulumi.com/docs/)
- [Azure Native Provider](https://www.pulumi.com/registry/packages/azure-native/)
- [Pulumi Examples](https://github.com/pulumi/examples)
- [Pulumi vs Terraform Comparison](https://www.pulumi.com/docs/intro/vs/terraform/)
- [Pulumi Blog](https://www.pulumi.com/blog/)
- [Pulumi Community Slack](https://slack.pulumi.com/)

---

Questions? Want to discuss IaC choices? Find me on [LinkedIn](https://linkedin.com/in/brianpsheridan) or [GitHub](https://github.com/two4suited)!
