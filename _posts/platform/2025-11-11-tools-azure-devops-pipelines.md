---
title: "Building a Modern Development Platform: Azure DevOps Pipeline Templates ♻️"
date: 2025-11-10T06:00:00-07:00
draft: false
categories: ["platform","azure-devops","cicd","pipelines","automation","templates"]
description: "Creating reusable Azure DevOps pipeline templates for standardized build, test, and deployment workflows across all platform applications"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

> 💻 **Source Code:** The complete template repository and pipeline examples are available in the [`aspire-tools-azure-devops`](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-azure-devops) branch of our GitHub repository.

## Introduction 🚀

In previous posts, we built individual tools: TypeSpec for API contracts, Kiota for client generation, Aspire for local orchestration, and Terraform for infrastructure. Now we need a way to consistently build, test, and deploy all these components across multiple projects. 

**Azure DevOps Pipeline Templates** solve this by centralizing all CI/CD logic in a shared repository. Instead of duplicating pipeline definitions across dozens of projects, teams define their pipelines once in a templates repository, then simply reference them with project-specific variables.

This approach ensures:
- ✅ **Consistency**: All projects follow the same build, test, and deploy patterns
- 🔄 **Maintainability**: Update the template once, all projects benefit immediately
- 🔒 **Security**: Centralized management of secrets, service connections, and deployment policies
- ⚡ **Efficiency**: Less boilerplate, faster pipeline setup for new projects

## What Are Pipeline Templates? 🤔

Pipeline templates are reusable YAML files that define jobs, steps, or entire pipelines. They abstract away implementation details and expose only the parameters needed for customization.

**Key Concepts:**

- 📋 **Template Repository**: Central repo containing all pipeline logic
- 🔧 **Variables File**: Project-specific variables (vars.yml) passed to templates
- 📦 **App-Specific Libraries**: Azure DevOps Library Groups for environment-specific secrets and configs
- 🎯 **Extends Pattern**: Child pipelines extend templates using the `extends` keyword
- 🔀 **Reusability**: Templates can be used across multiple projects and repositories

## Architecture: Templates Repository 🏗️

A typical templates repository structure looks like this:

```
pipeline-templates/
├── README.md
├── azure-pipelines.yml           # Main entry point (rarely used)
├── stages/
│   ├── build.yml                 # Build stage template
│   ├── test.yml                  # Test stage template
│   └── deploy.yml                # Deploy stage template
├── jobs/
│   ├── build-dotnet.yml          # .NET build job
│   ├── build-typescript.yml      # TypeScript build job
│   ├── test-unit.yml             # Unit testing job
│   └── test-integration.yml      # Integration testing job
├── steps/
│   ├── restore-nuget.yml         # NuGet restore step
│   ├── build-app.yml             # App build step
│   ├── publish-artifact.yml      # Artifact publishing
│   └── deploy-terraform.yml      # Terraform deployment
└── variables/
    └── default-vars.yml          # Default variable definitions
```

## How It Works: The Extends Pattern 🔄

The magic of template reusability comes from the `extends` keyword. Here's how it works:

**Step 1: Define a Template (templates-repo/stages/build.yml)**
```yaml
stages:
  - stage: Build
    displayName: 'Build Application'
    jobs:
      - job: BuildJob
        displayName: 'Build'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: echo Building ${{ parameters.appName }}
            displayName: 'Build Step'
          - template: ../steps/build-app.yml
            parameters:
              buildConfiguration: ${{ parameters.buildConfiguration }}
              projectPath: ${{ parameters.projectPath }}
```

**Step 2: Create a vars.yml in Your Project**
```yaml
# vars.yml in your application repository
variables:
  appName: 'WeatherApp.Api'
  buildConfiguration: 'Release'
  projectPath: 'src/WeatherApp.Api'
  dockerRegistry: 'myregistry.azurecr.io'
  artifactFeedId: 'my-nuget-feed'
```

**Step 3: Extend the Template from Your Pipeline**
```yaml
# azure-pipelines.yml in your application repository
trigger:
  - main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: ${{ variables.appName }}
    buildConfiguration: ${{ variables.buildConfiguration }}
    projectPath: ${{ variables.projectPath }}
```

## Setting Up the Templates Repository 📦

### Step 1: Create the Repository 🗂️

Create a new Azure DevOps repository called `pipeline-templates`:

```bash
git clone https://dev.azure.com/yourorg/yourproject/_git/pipeline-templates
cd pipeline-templates
git checkout -b main
```

### Step 2: Define a Build Stage Template 🏗️

Create `stages/build.yml`:

```yaml
parameters:
  - name: appName
    type: string
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: projectPath
    type: string
  - name: dotnetVersion
    type: string
    default: '8.0'

stages:
  - stage: Build
    displayName: 'Build ${{ parameters.appName }}'
    jobs:
      - job: BuildDotNet
        displayName: 'Build .NET Application'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UseDotNet@2
            displayName: 'Install .NET ${{ parameters.dotnetVersion }}'
            inputs:
              version: ${{ parameters.dotnetVersion }}
              includePreviewVersions: false

          - task: DotNetCoreCLI@2
            displayName: 'Restore NuGet Packages'
            inputs:
              command: 'restore'
              projects: '${{ parameters.projectPath }}/**/*.csproj'

          - task: DotNetCoreCLI@2
            displayName: 'Build Application'
            inputs:
              command: 'build'
              projects: '${{ parameters.projectPath }}/**/*.csproj'
              arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'

          - task: DotNetCoreCLI@2
            displayName: 'Publish Artifact'
            inputs:
              command: 'publish'
              publishWebProjects: false
              projects: '${{ parameters.projectPath }}/**/*.csproj'
              arguments: '--configuration ${{ parameters.buildConfiguration }} --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true

          - task: PublishBuildArtifacts@1
            displayName: 'Publish Build Artifact'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'drop'
```

### Step 3: Define a Test Stage Template ✅

Create `stages/test.yml`:

```yaml
parameters:
  - name: projectPath
    type: string
  - name: testProjectPattern
    type: string
    default: '**/*Tests.csproj'

stages:
  - stage: Test
    displayName: 'Run Tests'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - job: UnitTests
        displayName: 'Unit Tests'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - checkout: self
          - task: UseDotNet@2
            displayName: 'Install .NET'
            inputs:
              version: '8.0'

          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              projects: '${{ parameters.projectPath }}/${{ parameters.testProjectPattern }}'
              arguments: '--no-build --logger trx --collect:"XPlat Code Coverage"'

          - task: PublishTestResults@2
            condition: succeededOrFailed()
            inputs:
              testResultsFormat: 'VSTest'
              testResultsFiles: '**/*.trx'
              searchFolder: '$(Agent.TempDirectory)'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '$(Agent.TempDirectory)/**/*coverage.cobertura.xml'
```

### Step 4: Define a Deploy Stage Template 🚀

Create `stages/deploy.yml`:

```yaml
parameters:
  - name: environment
    type: string
    values:
      - 'dev'
      - 'staging'
      - 'production'
  - name: appName
    type: string
  - name: resourceGroup
    type: string
  - name: appServiceName
    type: string

stages:
  - stage: Deploy_${{ parameters.environment }}
    displayName: 'Deploy to ${{ parameters.environment }}'
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        displayName: 'Deploy ${{ parameters.appName }}'
        environment: ${{ parameters.environment }}
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop

                - task: AzureWebApp@1
                  displayName: 'Deploy to App Service'
                  inputs:
                    azureSubscription: '$(serviceConnectionName)'
                    appType: 'webAppLinux'
                    appName: '${{ parameters.appServiceName }}'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                    runtimeStack: 'DOTNETCORE|8.0'
```

## Using Templates in Your Project 📋

### Step 1: Reference the Templates Repository 🔗

In your application's `azure-pipelines.yml`:

```yaml
trigger:
  - main

resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/main

extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'
    buildConfiguration: 'Release'
    projectPath: 'src/WeatherApp.Api'
```

### Step 2: Create vars.yml for Variables 📝

Create `vars.yml` in your project root:

```yaml
variables:
  # Application Information
  appName: 'WeatherApp.Api'
  displayName: 'Weather API'
  
  # Build Configuration
  buildConfiguration: 'Release'
  dotnetVersion: '8.0'
  projectPath: 'src/WeatherApp.Api'
  
  # Deployment Settings
  appServiceName: 'weather-api-dev'
  resourceGroup: 'platform-rg-dev'
  
  # Docker & Registry
  dockerRegistry: 'myregistry.azurecr.io'
  dockerImageName: 'weatherapp/api'
  
  # Azure DevOps Library Group
  libraryGroupName: 'weather-app-secrets'
```

### Step 3: Link to Library Groups 🔐

In your pipeline, reference Azure DevOps Library Groups for app-specific secrets:

```yaml
extends:
  template: stages/build.yml@templates
  parameters:
    appName: 'WeatherApp.Api'

variables:
  - group: $(libraryGroupName)  # References 'weather-app-secrets' library
  - template: vars.yml
```

**Library Group: weather-app-secrets**
```
ContainerRegistry.Username
ContainerRegistry.Password
CosmosDb.ConnectionString
AppInsights.InstrumentationKey
ServiceBus.ConnectionString
```

## Advanced: Multi-Stage Pipeline with Extends 🔄

Chain multiple templates together for a complete CI/CD workflow:

```yaml
trigger:
  - main

resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/heads/main

variables:
  - template: vars.yml

extends:
  template: stages/build.yml@templates
  parameters:
    appName: ${{ variables.appName }}
    buildConfiguration: ${{ variables.buildConfiguration }}
    projectPath: ${{ variables.projectPath }}
    dotnetVersion: ${{ variables.dotnetVersion }}

stages:
  - template: stages/test.yml@templates
    parameters:
      projectPath: ${{ variables.projectPath }}

  - template: stages/deploy.yml@templates
    parameters:
      environment: 'dev'
      appName: ${{ variables.appName }}
      resourceGroup: ${{ variables.resourceGroup }}
      appServiceName: ${{ variables.appServiceName }}
```

## Benefits of This Approach ✨

### 1. **Consistency Across Projects** 🎯
- All projects follow the same build, test, and deploy patterns
- Standard naming conventions and artifact management
- Unified logging and monitoring

### 2. **DRY (Don't Repeat Yourself)** 🧹
- Define once, use everywhere
- Bug fixes in templates benefit all projects immediately
- No copy-paste pipeline definitions

### 3. **Security** 🔒
- Centralized management of secrets and service connections
- Controlled access to deployment environments
- Audit trail for all pipeline changes

### 4. **Scalability** 📈
- Add new projects with minimal pipeline setup
- Templates grow with your platform
- Easy to onboard new teams

### 5. **Flexibility** 🔄
- Templates support parameters for project-specific customization
- Library groups allow environment-specific configurations
- Can mix standard and custom steps

## Best Practices 🎯

### 1. **Organize Templates Logically** 📚

```
templates/
├── stages/          # Complete stages (build, test, deploy)
├── jobs/            # Reusable jobs
├── steps/           # Reusable steps
└── variables/       # Common variable definitions
```

### 2. **Use Meaningful Parameter Names** 📝

```yaml
parameters:
  - name: appName          # ✅ Clear and descriptive
    type: string
  - name: bc               # ❌ Avoid abbreviations
    type: string
```

### 3. **Document Template Parameters** 📖

```yaml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'
    displayName: 'Build Configuration'
    values:
      - Debug
      - Release
```

### 4. **Version Your Templates** 🏷️

Pinning your templates to specific versions prevents unexpected breaking changes when the templates repository is updated. Use Git tags to mark releases:

**Creating a Release Tag:**
```bash
cd pipeline-templates
git tag -a v1.0.0 -m "Release v1.0.0: Initial build, test, deploy stages"
git push origin v1.0.0
```

**Referencing a Specific Version in Your Project:**
```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates
      ref: refs/tags/v1.0.0  # ✅ Pin to specific version
```

**Benefits of Version Pinning:**

- 🔒 **Stability**: Your pipelines don't break when templates change
- 📚 **Traceability**: Know exactly which template version each project uses
- 🔄 **Controlled Upgrades**: Deliberately upgrade when ready, not automatically
- 🎯 **Rollback**: Easy to revert to previous template versions if issues arise

**Semantic Versioning Strategy:**

```
v1.0.0  - MAJOR.MINOR.PATCH

MAJOR: Breaking changes (parameter names changed, stage structure modified)
MINOR: New features (new optional parameters, new templates added)
PATCH: Bug fixes (step logic corrected, typos fixed)

Examples:
v1.0.0  → Initial release
v1.1.0  → Add new optional parameters for logging
v1.1.1  → Fix bug in test stage
v2.0.0  → Restructure deploy stages (breaking change)
```

**Version Management Workflow:**

1. **Development**: Use `main` branch for ongoing work
   ```yaml
   ref: refs/heads/main  # Only use for testing/development
   ```

2. **Testing**: Create release candidate tags
   ```yaml
   ref: refs/tags/v1.1.0-rc1
   ```

3. **Production**: Pin to stable releases
   ```yaml
   ref: refs/tags/v1.1.0
   ```

**Template Repository README with Versions:**

```markdown
# Pipeline Templates

Latest Release: v1.1.0

## Version History

### v1.1.0 (2025-11-11)
- Add support for TypeScript build jobs
- New optional parameter: `eslintEnabled`
- Fix: Test stage now properly handles skip conditions

### v1.0.0 (2025-11-04)
- Initial release
- Build stage for .NET applications
- Test stage with unit test support
- Deploy stage for Azure App Service

## Migration Guide

### Upgrading from v1.0.0 to v1.1.0
No breaking changes. Existing projects will continue to work.
```

### 5. **Use Library Groups for Secrets** 🔐

Never hardcode secrets in YAML:

```yaml
# ❌ Don't do this
variables:
  apiKey: 'supersecret123'

# ✅ Do this instead
variables:
  - group: my-app-secrets
```

## Troubleshooting 🔧

### Issue: Template Not Found

**Error:**
```
Resource not found: 'stages/build.yml'
```

**Solution:** Ensure the repository resource is correctly defined:
```yaml
resources:
  repositories:
    - repository: templates
      type: git
      name: YourProject/pipeline-templates  # Correct format: Project/Repo
```

### Issue: Parameters Not Substituting

**Error:**
```
${{ parameters.appName }} appears literally in output
```

**Solution:** Use the `extends` keyword at the pipeline root level, not in stages.

### Issue: Library Group Not Found

**Error:**
```
The variable group was not found or not authorized.
```

**Solution:** Ensure the pipeline has permissions to access the library group through the project's pipeline settings.

## Conclusion 🎉

Pipeline templates are a game-changer for platform teams. Instead of managing dozens of independent pipelines, you maintain a single source of truth. Projects become simpler, more consistent, and easier to maintain.

The combination of:
- ✅ Centralized pipeline templates
- ✅ Project-specific vars.yml files
- ✅ Azure DevOps library groups
- ✅ The extends pattern

Creates a powerful, scalable approach to CI/CD that grows with your organization.

**Key Takeaways:**
1. Store all pipeline logic in a `pipeline-templates` repository
2. Use the `extends` keyword to include templates from projects
3. Pass project-specific variables via `vars.yml` and parameters
4. Use Azure DevOps library groups for environment-specific secrets
5. Version your templates and pin to specific releases in projects

Start building your templates repository today, and watch your CI/CD practices transform!

---

**Resources:** 📚

- [Azure Pipelines Template Syntax](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/pipeline)
- [Using Template Expressions](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/template-expressions)
- [Variable Groups and Library](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups)
- [Pipeline Resources](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema/resources)
- [Our GitHub Repository](https://github.com/two4suited/blog-platform-aspire)
