---
title: "Building a Modern Development Platform: Creating Custom dotnet new Templates 🏗️"
date: 2025-11-15T12:00:00-07:00
draft: false
categories: ["platform","dotnet","templates","automation","tooling"]
description: "Build reusable .NET templates with dotnet new to standardize project structure and accelerate development across your organization"
toc: true
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Why Custom Templates? 💡

Every organization has standards:
- ✅ Consistent project structure
- ✅ Standard dependencies and NuGet packages
- ✅ Configuration patterns (logging, health checks, observability)
- ✅ Service defaults and resilience handlers
- ✅ README templates and documentation

**Without templates:**
```
Developer A creates new project
├── Picks own folder structure
├── Uses different dependency versions
├── Sets up logging differently
└── Everyone copies-paste templates anyway

Developer B creates new project
├── Different folder structure
├── Different dependency versions
├── Different logging setup
└── Inconsistent codebase
```

**With custom templates:**
```
dotnet new mycompanytemplate -n MyProject
└── Instantly generates standardized, production-ready project
    ├── ✅ Correct folder structure
    ├── ✅ Standard dependencies
    ├── ✅ Preconfigured logging and observability
    ├── ✅ Service defaults for resilience
    └── ✅ Ready to run
```

## How Templates Work 🔧

The `dotnet new` system consists of two key parts:

### 1. **Template Definition** 📋

```
mytemplate/
├── .template.config/
│   └── template.json          ← Configuration
├── src/
│   └── MyApp.csproj           ← Template files
├── README.md
└── global.json
```

The `.template.config/template.json` file tells the CLI:
- What this template is called
- What parameters it accepts
- How to handle conditional file inclusion/exclusion
- What placeholder names to replace

### 2. **Installation** 📦

Templates can be installed from:
- **NuGet package** (published to nuget.org or private feed)
- **Local directory** (for development)
- **Nupkg file** (for testing before publishing)

Once installed, use it like any built-in template:
```bash
dotnet new mytemplate -n MyProject
```

## The Minimum Template Configuration 📄

Here's the absolute minimum you need:

```
mytemplate/
├── .template.config/
│   └── template.json
└── yourfiles/
    ├── file.cs
    └── README.md
```

The `template.json` file:

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Your Company",
  "name": "My Template",
  "shortName": "mytemplate",
  "identity": "Company.MyTemplate.CSharp",
  "classifications": ["Common", "Library"],
  "sourceName": "MyTemplate"
}
```

**Key Fields:**

| Field | Purpose | Example |
|-------|---------|---------|
| `$schema` | JSON schema URL for editor IntelliSense | http://json.schemastore.org/template |
| `author` | Who created this | "Your Company" |
| `name` | Display name | "My Awesome Template" |
| `shortName` | Used in `dotnet new` command | "mytemplate" |
| `identity` | Unique identifier (reverse domain format) | "Company.MyTemplate.CSharp" |
| `classifications` | Tags for searching/filtering | ["Common", "Console"] |
| `sourceName` | Text to replace with user's project name | "MyTemplate" |

When a user runs:
```bash
dotnet new mytemplate -n MyAwesomeApp
```

The template engine:
1. ✅ Finds all occurrences of `MyTemplate` in filenames and content
2. ✅ Replaces them with `MyAwesomeApp`
3. ✅ Generates the project

## A Real-World Example: .NET Aspire Template 🚀

Let me show you a complete, practical template for creating cloud-native solutions with .NET Aspire:

### Template Structure

```
template/
├── Template.csproj                    # Packaging project
└── NetSolution/                       # Template content
    ├── .template.config/
    │   └── template.json              # Configuration
    ├── NetSolution.slnx               # Solution file
    ├── apphost.cs                     # Single-file Aspire AppHost
    ├── global.json
    ├── .gitignore
    ├── README.md
    └── src/
        ├── NetSolution.Api/           # Minimal API project
        │   ├── Program.cs
        │   ├── Health.cs
        │   └── NetSolution.Api.csproj
        ├── NetSolution.AppHost/       # Aspire host
        │   ├── apphost.cs
        │   └── NetSolution.AppHost.csproj
        ├── NetSolution.ServiceDefaults/
        │   ├── Extensions.cs
        │   └── NetSolution.ServiceDefaults.csproj
        └── NetSolution.Data/          # Conditional: only if --IncludeCosmos
            ├── CosmosContext.cs
            ├── Models/
            └── NetSolution.Data.csproj
```

### The template.json Configuration

```json
{
  "$schema": "http://json.schemastore.org/template",
  "author": "Your Company",
  "name": ".NET Solution Template",
  "description": "Create a complete .NET Aspire solution with optional Cosmos DB",
  "shortName": "netsolution",
  "identity": "Company.NetSolution.Template",
  "classifications": ["Common", "Cloud", "Aspire"],
  "sourceName": "NetSolution",
  
  "symbols": {
    "IncludeCosmos": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "false",
      "displayName": "Include Cosmos DB",
      "description": "Add Entity Framework Core with Cosmos DB support"
    }
  },
  
  "sources": [
    {
      "modifiers": [
        {
          "condition": "(!IncludeCosmos)",
          "exclude": [
            "src/NetSolution.Data/**/*"
          ]
        }
      ]
    }
  ]
}
```

**What this does:**
- ✅ Defines one parameter: `IncludeCosmos` (boolean, defaults to false)
- ✅ Excludes the Data project folder if `IncludeCosmos` is false
- ✅ Allows conditional project generation

### Using Conditional Compilation in Code

In your C# files, use preprocessor directives:

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

builder.Services.AddMinimalApi();

#if (IncludeCosmos)
builder.Services.AddCosmosDbContext();
#endif

var app = builder.Build();
// ...
```

For XML files (csproj, slnx), use comment syntax:

```xml
<!--#if (IncludeCosmos)-->
<ItemGroup>
  <ProjectReference Include="..\NetSolution.Data\NetSolution.Data.csproj" />
</ItemGroup>
<!--#endif-->
```

## Packaging Your Template 📦

### Step 1: Create a Packaging Project

Create `Template.csproj` at the template root:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <PackageType>Template</PackageType>
    <PackageVersion>1.0.0</PackageVersion>
    <PackageId>Company.NetSolution.Template</PackageId>
    <Title>Company .NET Solution Template</Title>
    <Authors>Your Company</Authors>
    <Description>Complete .NET Aspire solution with optional Cosmos DB</Description>
    <PackageTags>dotnet-new;templates;aspire;cloud-native</PackageTags>
    <TargetFramework>netstandard2.0</TargetFramework>

    <IncludeContentInPack>true</IncludeContentInPack>
    <IncludeBuildOutput>false</IncludeBuildOutput>
    <ContentTargetFolders>content</ContentTargetFolders>
  </PropertyGroup>

  <ItemGroup>
    <Content Include="NetSolution\**\*" 
             Exclude="NetSolution\**\bin\**;NetSolution\**\obj\**" />
    <Compile Remove="**\*" />
  </ItemGroup>
</Project>
```

**Key Settings:**
- `<PackageType>Template</PackageType>` - Identifies this as a template package
- `<IncludeContentInPack>true</IncludeContentInPack>` - Include all template files
- `<IncludeBuildOutput>false</IncludeBuildOutput>` - Don't include binaries
- `<ContentTargetFolders>content</ContentTargetFolders>` - Put templates in content folder
- `<Compile Remove="**\*" />` - Don't compile template code

### Step 2: Pack the Template

```bash
cd template
dotnet pack
```

This creates `bin/Release/Company.NetSolution.Template.1.0.0.nupkg`

### Step 3: Install and Test

```bash
# Install from local nupkg
dotnet new install ./bin/Release/Company.NetSolution.Template.1.0.0.nupkg

# Verify installation
dotnet new list

# Create a test project
mkdir test-output
cd test-output
dotnet new netsolution -n TestApp --IncludeCosmos
cd TestApp/src/TestApp.AppHost
dotnet run
```

## Installing Templates 🔗

### From Local Directory (Development)

```bash
# Install from folder
dotnet new install ./path/to/template/NetSolution

# Later, uninstall
dotnet new uninstall ./path/to/template/NetSolution
```

**Use case:** Developing/testing templates locally

### From NuGet Package

```bash
# Install specific version
dotnet new install Company.NetSolution.Template

# Install latest version
dotnet new install Company.NetSolution.Template::latest

# List installed templates
dotnet new list

# Get template details
dotnet new netsolution --help
```

### From Custom NuGet Feed

```bash
dotnet new install Company.NetSolution.Template \
  --nuget-source https://your-nuget-feed.com/v3/index.json
```

## Using the Template 🚀

### Basic Usage

```bash
# Create project without optional features
dotnet new netsolution -n MyAwesomeApp
cd MyAwesomeApp
```

### With Parameters

```bash
# Enable Cosmos DB support
dotnet new netsolution -n MyAwesomeApp --IncludeCosmos

# Alternative syntax
dotnet new netsolution -n MyAwesomeApp --IncludeCosmos true
```

### View Available Parameters

```bash
dotnet new netsolution --help
```

Output:
```
Options:
  -n, --name <name>          The name for the output being created. If no name
                             is specified, the name of the current directory is
                             used.
  --IncludeCosmos            Include Cosmos DB support with Data project
                             (default: false)
```

## Advanced Configuration 🎯

### Adding More Parameters

Edit `template.json` and add to the `symbols` section:

```json
{
  "symbols": {
    "IncludeCosmos": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "false",
      "displayName": "Include Cosmos DB",
      "description": "Add Entity Framework Core with Cosmos DB support"
    },
    "IncludeAuth": {
      "type": "parameter",
      "datatype": "bool",
      "defaultValue": "false",
      "displayName": "Include Authentication",
      "description": "Add OpenID Connect/OAuth2 authentication"
    },
    "ApiVersion": {
      "type": "parameter",
      "datatype": "text",
      "defaultValue": "v1",
      "displayName": "API Version",
      "description": "API version for generated endpoints",
      "replaces": "API_VERSION"
    }
  }
}
```

### Parameter Types

| Type | Usage | Example |
|------|-------|---------|
| `bool` | Yes/no options | `--IncludeCosmos` |
| `text` | String replacement | `--ApiVersion v2` |
| `choice` | Select from options | `--Framework net8.0` |

### Conditional File Exclusion

```json
{
  "sources": [
    {
      "modifiers": [
        {
          "condition": "(!IncludeCosmos)",
          "exclude": [
            "src/NetSolution.Data/**/*"
          ]
        },
        {
          "condition": "(!IncludeAuth)",
          "exclude": [
            "src/NetSolution.Api/Auth/**/*"
          ]
        }
      ]
    }
  ]
}
```

### Replace Patterns

Use `replaces` to substitute text throughout the template:

```json
{
  "symbols": {
    "ApiVersion": {
      "type": "parameter",
      "datatype": "text",
      "defaultValue": "v1",
      "replaces": "API_VERSION"
    }
  }
}
```

Then in your code:

```csharp
const string API_VERSION = "API_VERSION";

public class Api
{
    public string GetVersion() => API_VERSION;
}
```

When generated with `--ApiVersion v2`, the constant becomes:

```csharp
const string API_VERSION = "v2";
```

## Publishing to NuGet 📤

### Step 1: Create NuGet.org Account

1. Go to [nuget.org](https://www.nuget.org/)
2. Sign up or sign in
3. Create an API key in your account settings

### Step 2: Pack and Push

```bash
# Pack the template
dotnet pack

# Push to nuget.org
dotnet nuget push ./bin/Release/Company.NetSolution.Template.1.0.0.nupkg \
  --api-key YOUR_API_KEY \
  --source https://api.nuget.org/v3/index.json
```

### Step 3: Install from NuGet

Once published, anyone can install:

```bash
dotnet new install Company.NetSolution.Template
dotnet new netsolution -n MyProject
```

## Best Practices ✅

### 1. **Keep Templates Up-to-Date** 🔄

Your template should reflect current best practices:
- ✅ Use latest .NET LTS or current version
- ✅ Include modern packages (Aspire, minimal APIs, etc.)
- ✅ Add new observability/health check patterns
- ✅ Remove deprecated packages

Bump the version when updating:
```xml
<PackageVersion>1.1.0</PackageVersion>
```

### 2. **Include Comprehensive Documentation** 📖

In your template's README:
```markdown
# MyTemplate

## What Gets Generated
- Aspire orchestration
- Minimal API
- Health checks
- Service defaults

## Installation
```bash
dotnet new install Company.MyTemplate
```

## Usage
```bash
dotnet new mytemplate -n MyProject --Feature1
```

## Parameters
- `--Feature1`: Enable feature 1
```

### 3. **Test With Real Projects** 🧪

Before publishing:
```bash
# Install locally
dotnet new install ./NetSolution

# Create multiple test projects
dotnet new netsolution -n TestApp1
dotnet new netsolution -n TestApp2 --IncludeCosmos

# Verify each builds and runs
cd TestApp1 && dotnet build
cd TestApp2 && dotnet run
```

### 4. **Use Semantic Versioning** 📊

```
1.0.0  Initial release
1.1.0  New feature added (backward compatible)
1.1.1  Bug fix
2.0.0  Breaking changes (new major version)
```

### 5. **Include Helpful Examples** 💡

In generated files, include comments showing how to use generated code:

```csharp
// Example: Add a new endpoint
// app.MapGet("/items/{id}", GetItemById);
//
// private static async Task<Item> GetItemById(string id, CosmosContext db)
// {
//     return await db.Items.FindAsync(id);
// }

app.MapHealthChecks("/health");
```

### 6. **Validate Parameter Names** ⚠️

Parameter names in C# preprocessing are **case-sensitive**:

```csharp
#if (IncludeCosmos)      // ✅ Correct - matches symbol name
#endif

#if (includecosmos)      // ❌ Wrong - won't work
#endif
```

### 7. **Keep Templates Focused** 🎯

One template = one clear purpose:
- ✅ `netsolution` - Complete Aspire solutions
- ✅ `netlibrary` - Reusable class libraries
- ✅ `networker` - Background job services

Don't try to handle every scenario in one template.

## Troubleshooting 🔧

### Template Not Found After Installation

```bash
# Try the full path
dotnet new install /full/path/to/template/NetSolution

# Verify it's installed
dotnet new list | grep netsolution
```

### Conditional Content Not Working

✅ C# files: Use `#if (ParameterName)` without quotes
```csharp
#if (IncludeCosmos)
// ...
#endif
```

✅ XML files: Use comment syntax
```xml
<!--#if (IncludeCosmos)-->
<ItemGroup>
  <!-- ... -->
</ItemGroup>
<!--#endif-->
```

❌ Case sensitivity matters - `IncludeCosmos` ≠ `includecosmos`

### Files Not Being Excluded

Verify the condition syntax in `template.json`:
```json
{
  "condition": "(!IncludeCosmos)",  // Parentheses and ! are required
  "exclude": [
    "src/NetSolution.Data/**/*"
  ]
}
```

### NuGet Install Issues

Clear the cache and retry:
```bash
dotnet nuget locals all --clear
dotnet new install Company.MyTemplate
```

## Resources 📚

- **Official Docs**: [Custom templates for dotnet new](https://learn.microsoft.com/en-us/dotnet/core/tools/custom-templates)
- **Examples**: [dotnet/templating samples](https://github.com/dotnet/templating/tree/main/template_feed/Microsoft.DotNet.Web.ItemTemplates/templates)
- **Wiki**: [dotnet/templating GitHub Wiki](https://github.com/dotnet/templating/wiki)
- **Blog**: [How to create your own templates for dotnet new](https://devblogs.microsoft.com/dotnet/how-to-create-your-own-templates-for-dotnet-new/)
- **Schema**: [template.json JSON Schema](http://json.schemastore.org/template)

## Conclusion 🎉

Custom `dotnet new` templates standardize your organization's development:

**Benefits:**
- ✅ Consistency across all new projects
- ✅ Reduced onboarding time for new developers
- ✅ Best practices baked into every project
- ✅ Easy to update and versioning control
- ✅ Community can benefit from your templates

**Next Steps:**
1. 🎯 Identify common patterns in your organization
2. 🏗️ Create a template capturing those patterns
3. 📦 Package and publish to NuGet
4. 📢 Share with your team
5. 🔄 Maintain and improve over time

Your developers will thank you for making project creation instant and consistent!

---

**Resources:** 📚

- [Example Template Repository](https://github.com/two4suited/blog-net-template)
- [Custom templates for dotnet new](https://learn.microsoft.com/en-us/dotnet/core/tools/custom-templates)
- [.NET Aspire Documentation](https://learn.microsoft.com/en-us/dotnet/aspire/)
- [Entity Framework Core](https://learn.microsoft.com/en-us/ef/core/)
- [dotnet/templating Wiki](https://github.com/dotnet/templating/wiki)
