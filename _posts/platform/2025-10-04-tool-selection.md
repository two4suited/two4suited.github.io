---
title: "Building a Modern Development Platform: Tool Selection 🛠️"
date: 2025-10-04T06:00:00-07:00
draft: false
categories: ["platform","aspire","typespec","kiota","dotnet","terraform","modernization","cloud"]
description: "A journey from legacy on-premise .NET Framework applications to a modern, cloud-native platform with standardized tooling, IaC, and developer experience"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

In our [previous post](https://brianpsheridan.com/platform/aspire/typespec/kiota/dotnet/terraform/modernization/cloud/2025/10/03/building-modern.html), we outlined the vision for our modern development platform. Now comes the critical question: which tools would form the foundation of this transformation? The choices we made needed to work together cohesively while providing a clear migration path from our legacy .NET Framework applications.

## The Tool Selection Process 🎯

Our approach to tool selection was pragmatic rather than aspirational. We weren't looking for the newest, shiniest technologies – we needed proven tools that would:
- 🔗 Work together as an integrated ecosystem
- 🏢 Support our existing Microsoft-heavy technology stack
- 🛣️ Provide a reasonable migration path from .NET Framework
- 🤝 Offer strong community support and long-term viability
- ✅ Solve real problems we were experiencing today

Let's dive into each decision and the reasoning behind it.

## Cloud Environment: Azure ☁️

**The Problem**: We needed to move from on-premise infrastructure to the cloud, but which cloud provider?

**The Solution**: We chose **Microsoft Azure** as our cloud platform.

**Why Azure?** As a government agency deeply invested in the Microsoft ecosystem, Azure was the natural choice. Our existing expertise with Microsoft technologies provided a smoother learning curve than switching to AWS or GCP. More importantly, Azure's offerings aligned perfectly with our needs:
- 🔐 **Azure API Management** provided enterprise-grade API gateway capabilities with built-in security, rate limiting, and policy enforcement
- 🖥️ **App Service Plans** offered a managed platform for hosting our applications without the complexity of managing underlying infrastructure
- 🔑 **Entra ID** integration simplified our authentication and authorization requirements
- 🤝 **Existing Microsoft partnerships** and support made adoption smoother

While AWS might offer more services, Azure's integration with our existing Microsoft stack made it the clear winner for our use case.

## Local Development: .NET Aspire 💻

**The Problem**: Local development was a nightmare. Developers needed complex environment setups, database connections, and multiple services running simultaneously just to work on a single feature.

**The Solution**: We adopted **.NET Aspire** for orchestrating local development environments.

**Why Aspire?** Aspire is a relatively new project from Microsoft, but it's backed by some serious talent from the .NET team. It solves a problem we were struggling with: how do you run a distributed application locally without losing your mind? 

Aspire provides:
- 🔄 **Service orchestration** that makes it trivial to run multiple services, databases, and dependencies locally
- 🔍 **Service discovery** built-in, so services can find each other without hardcoded configuration
- 📊 **Telemetry and observability** out of the box with OpenTelemetry integration
- 🚀 **Future potential** as an Infrastructure as Code (IaC) and deployment tool – the Aspire team is actively working on cloud deployment capabilities

What really sold us was the developer experience. With Aspire, a developer can clone a repo, hit F5, and have the entire application stack running locally with proper service-to-service communication, databases, and monitoring. No more "it works on my machine" excuses.

The fact that Aspire is evolving to handle deployment to Azure Container Apps means our investment today will pay dividends tomorrow when we need to deploy these same applications to the cloud.

## API Contract Generation: TypeSpec 📋

**The Problem**: Teams were building APIs with inconsistent structures, documentation lagged behind implementation, and client teams couldn't start development until the API was mostly complete.

**The Solution**: We adopted **TypeSpec** (formerly known as Cadl) for API contract definition.

**Why TypeSpec?** TypeSpec represents the future of API specification. While OpenAPI (Swagger) is the industry standard, writing OpenAPI by hand is tedious and error-prone. TypeSpec provides:
- ✍️ **Clean, readable syntax** that's easier to write and maintain than raw OpenAPI YAML
- 🔒 **Type safety** and validation at the specification level
- 📤 **OpenAPI emission** – TypeSpec can generate OpenAPI 3.0 specs, so we get the best of both worlds
- 🤖 **Code generation** capabilities for both clients and servers
- 🏢 **Microsoft backing** with active development and growing ecosystem

By defining our APIs in TypeSpec first, we enforce a contract-first development approach. The API contract becomes the source of truth, and both client and server implementations are generated from that contract. This eliminates the drift between documentation and implementation that plagued our legacy systems.

## API Client Code Generation: Kiota 🔧

**The Problem**: While TypeSpec can generate client code, we needed more flexibility, especially around authentication patterns that varied across our services.

**The Solution**: We chose **Kiota** for generating API client SDKs.

**Why Kiota?** Kiota is Microsoft's next-generation API client generator that produces idiomatic code for multiple languages (C#, TypeScript, Java, Python, Go, and more). What sets Kiota apart:
- 🔓 **Authentication flexibility** – Kiota makes it easy to plug in different authentication providers (OAuth, API keys, custom headers)
- ⚡ **Minimal dependencies** – Generated clients are lightweight with only necessary dependencies
- 📱 **Native async/await patterns** that feel natural in modern .NET and TypeScript
- 📄 **OpenAPI 3.0 support** – Works perfectly with the OpenAPI specs generated by TypeSpec
- 🌍 **Multi-language support** – One spec, multiple client libraries for different teams

The combination of TypeSpec for contract definition and Kiota for client generation gives us a powerful workflow: define the API in TypeSpec, emit OpenAPI, generate clients with Kiota. We package these generated clients and publish them to our internal private NuGet feed, making it trivial for consuming teams to add a reference and start making type-safe API calls. Backend and frontend teams can work in parallel with confidence that the contract is being honored.

## Infrastructure as Code: Terraform 🏗️

**The Problem**: Manual infrastructure provisioning was slow, error-prone, and made it difficult to maintain consistency across environments.

**The Solution**: We standardized on **HashiCorp Terraform** with **Terraform Cloud** for state management and module hosting.

**Why Terraform?** While Azure has its own IaC solution (Bicep/ARM templates), Terraform's multi-cloud support and mature ecosystem made it the better choice:
- 🔌 **Provider ecosystem** – Terraform has providers for everything: Azure, Okta, GitHub, DNS, monitoring tools
- 📦 **Reusable modules** – We can create standardized modules for common patterns (API + database, worker service, etc.)
- 💾 **State management** – Terraform Cloud handles state storage, locking, and provides a UI for visualizing infrastructure
- 👁️ **Plan and Apply workflow** – The ability to preview changes before applying them reduces mistakes
- ☁️ **Multi-cloud flexibility** – If we ever need to use services from other clouds, our IaC patterns remain consistent

Terraform Cloud's private module registry allows us to publish internal modules that teams can consume, ensuring everyone follows the same infrastructure patterns and security baselines.

## DevOps Platform: Azure DevOps ♻️

**The Problem**: We were already using Azure DevOps on-premise, but it was outdated and lacked the modern CI/CD capabilities we needed.

**The Solution**: We migrated to **Azure DevOps Cloud** and created standardized pipeline templates.

**Why Azure DevOps Cloud?** We considered GitHub Actions, but Azure DevOps provided better support for our specific needs:
- 📋 **Enterprise features** we were already using (work item tracking, test management)
- 📝 **Pipeline templates** allow us to create reusable, standardized build and deploy workflows
- 📦 **Artifact feeds** for hosting our NuGet packages and npm modules
- 🔐 **Variable groups** and **service connections** for managing secrets and cloud credentials
- 💡 **Existing organizational knowledge** – our teams already knew Azure DevOps

By creating pipeline templates, we ensure every application follows the same build, test, and deployment process. Developers don't need to be CI/CD experts – they just reference the template and provide a few parameters.

## Application Templates: .NET Core Templating Engine 📦

**The Problem**: Each team was creating new projects from scratch, leading to inconsistent structure, missing security configurations, and duplicated effort.

**The Solution**: We leveraged the **.NET Core templating engine** to create standardized project templates.

**Why .NET Templates?** The .NET templating system is powerful and often overlooked:
- 🔧 **Built into the .NET SDK** – No additional tools required
- 🎛️ **Flexible parameterization** – Templates can accept parameters to customize the generated code
- 📁 **Multi-file templates** – Can generate entire project structures with proper folder organization
- ❓ **Conditional content** – Include/exclude files based on template parameters
- ⚙️ **Post-actions** – Run commands after template generation (restore packages, initialize git, etc.)

We created comprehensive templates for common scenarios that include everything needed to run on our platform:
- 🔌 **API with Database Backend** – .NET API with TypeSpec contract, Entity Framework, Terraform infrastructure, and Aspire orchestration
- ⚛️ **API with React Frontend** – Full-stack application with .NET backend, React TypeScript frontend, shared infrastructure, and local dev setup
- ⏰ **Background Worker Service** – .NET worker with infrastructure provisioning and Aspire integration
- ⚡ **Azure Function** – Serverless function with dependency injection and Terraform deployment configuration

These templates don't just scaffold code – they include the complete platform integration: TypeSpec contracts, Terraform modules for infrastructure, Aspire AppHost configuration for local development, proper project structure, configuration management, logging, health checks, and our security baseline. A developer can run a single command and have a fully configured application ready to develop and deploy.

## Developer Tooling: Custom CLI 🖥️

**The Problem**: Even with templates, developers needed to install multiple tools, configure their environment, and remember various commands to get started.

**The Solution**: We built a **custom CLI tool** that wraps common development tasks.

**Why Build Our Own CLI?** Our CLI, built in .NET Core, serves as a friendly wrapper around our platform:
- 🔨 **Environment setup** – Installs and configures necessary development tools (Docker, .NET SDK, Node.js, Terraform)
- 📋 **Template scaffolding** – Wraps the .NET templating engine with a user-friendly interface
- 🚀 **Project initialization** – Creates new projects with all necessary files, Azure DevOps pipelines, and git repositories
- 📤 **Deployment helpers** – Simplifies deploying applications to various environments
- ✅ **Configuration validation** – Checks that developer environments are properly configured

The CLI provides a consistent interface for developers, hiding complexity while maintaining the flexibility of the underlying tools. It's the "easy button" for our platform.

## Security: Okta + Terraform 🔒

**The Problem**: Security configuration was manual, inconsistent, and difficult to audit. Creating new auth servers, permissions, and integrations required tickets and days of waiting.

**The Solution**: We automated our **Okta** configuration using **Terraform's Okta provider**.

**Why Terraform for Okta?** We already had Okta as our identity provider, but managing it manually was painful:
- 🔌 **Okta Terraform provider** allows us to define auth servers, scopes, claims, groups, and applications as code
- 📚 **Version control** for security configurations – every change is tracked and reviewed
- 🤖 **Automated provisioning** – Creating a new API automatically creates its auth server configuration
- 📋 **Consistency** – Every API follows the same security patterns because they're generated from the same Terraform modules
- 📖 **Audit trail** – Git history shows who changed what security configuration and when

This automation reduced auth server setup from days to minutes and eliminated configuration errors.

## Service Virtualization: WireMock Cloud 🎭

**The Problem**: Testing integrations required all dependent services to be running, which was expensive in lower environments and impossible for external third-party services.

**The Solution**: We adopted **WireMock** and **WireMock Cloud** for service virtualization and mocking.

**Why WireMock?** WireMock is the industry-standard tool for API mocking:
- 📋 **Contract-based mocking** – Create mocks from OpenAPI specs (generated by TypeSpec!)
- ☁️ **WireMock Cloud** – Hosted solution from the creator of the open-source project, providing:
  - 📊 Centralized mock storage
  - 👥 Team collaboration on mock definitions
  - 🌍 Environment-specific mock configurations
  - 📈 Analytics on mock usage
- 🔄 **Stateful mocking** – Mocks can simulate complex workflows and state changes
- 🧪 **Integration testing** – Developers can test against mocks locally without external dependencies

This dramatically reduced our infrastructure costs (fewer environments needed) and improved developer productivity (no more waiting for downstream services to be available).

## Application Development: Framework Flexibility 🎨

**The Problem**: We needed to support modern application development while not forcing every team into the same technology choices.

**The Solution**: We adopted a **container-first, framework-agnostic** approach that provides flexibility while maintaining platform standards.

**Why Framework Flexibility?** While many of our applications currently use **.NET Core** for backends and **React with TypeScript** for frontends, we deliberately designed the platform to support any technology stack:

- 🐳 **Container-based deployment** – As long as an application can produce a Docker container, it can run on our platform
- 🔧 **Language agnostic** – Teams can choose the best tool for their specific problem domain
- 📋 **Common platform patterns** – Regardless of framework choice, all applications follow the same patterns for:
  - 📄 API contracts (TypeSpec/OpenAPI)
  - 🏗️ Infrastructure provisioning (Terraform)
  - 🖥️ Local development (Aspire orchestration)
  - 🔐 Authentication (Okta integration)
  - 📊 Observability (OpenTelemetry)
  - 🔄 CI/CD (Azure DevOps pipelines)

**Current Popular Choices**: While teams have flexibility, most are choosing:
- ⚙️ **.NET Core** for backends due to our existing expertise and migration path from .NET Framework
- ⚛️ **React with TypeScript** for frontends for type safety and integration with Kiota-generated clients
- 🐍 **Python** for data processing and ML workloads
- 📦 **Node.js** for specific microservices where the npm ecosystem provides advantages

The key insight: **standardize the platform, not the programming language**. By focusing on contracts, infrastructure patterns, and observability rather than specific frameworks, we give teams the flexibility they need while maintaining consistency where it matters.

## How It All Fits Together 🧩

These tools form an integrated ecosystem:

1. 👨‍💻 **Developer** uses the **CLI** to create a new API from a **template**
2. 📝 API contract is defined in **TypeSpec** and committed to git
3. 🔄 **Azure DevOps pipeline** builds the API, runs tests, and deploys to **Azure**
4. 🏗️ Infrastructure is provisioned via **Terraform** (App Service, database, API Management)
5. 🔐 Security is configured automatically in **Okta** via Terraform
6. 📚 **Kiota** generates client SDKs for frontend teams
7. 🖥️ **Aspire** orchestrates local development environment
8. 🎭 **WireMock** provides mocks for external dependencies
9. ⚛️ **React frontend** consumes the Kiota-generated client with type safety

Every tool was chosen to solve a specific problem while working harmoniously with the others. The result is a platform that accelerates development, enforces best practices, and provides a clear path forward from our legacy systems.

## What's Next 🔮

In upcoming posts, we'll dive deep into each of these tools:
- 🚀 Setting up .NET Aspire for local development
- 📝 Creating TypeSpec contracts and generating OpenAPI
- 🔧 Building Kiota clients with custom authentication
- 🏗️ Terraform patterns for Azure infrastructure
- 📦 Creating custom .NET templates
- 🎭 Service virtualization strategies with WireMock

Each tool deserves its own deep dive with practical examples from our real-world implementation.

Stay tuned! 🚀