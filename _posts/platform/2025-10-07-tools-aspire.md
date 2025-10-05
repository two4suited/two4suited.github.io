---
title: "Building a Modern Development Platform: .NET Aspire for Local Development 💻"
date: 2025-10-07T06:00:00-07:00
draft: false
categories: ["platform","aspire","dotnet","local-development","orchestration"]
description: "Setting up .NET Aspire for orchestrating local development environments with service discovery, telemetry, and seamless multi-service debugging"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction 🚀

In our [tool selection post](https://brianpsheridan.com/platform/aspire/typespec/kiota/dotnet/terraform/modernization/cloud/2025/10/04/tool-selection.html), we introduced .NET Aspire as our solution for orchestrating local development environments. Today, we're going to dive deep into Aspire by building a real-world application from scratch.

By the end of this post, you'll have a complete understanding of how Aspire simplifies local development for distributed applications. We'll build a full-stack application with:
- A .NET Web API backed by Azure Cosmos DB
- A React TypeScript frontend
- Aspire orchestration for running everything locally
- Built-in observability with the Aspire Dashboard

**What is .NET Aspire?** Aspire is an opinionated, cloud-ready stack for building observable, production-ready distributed applications. It handles service orchestration, service discovery, and telemetry out of the box, making local development of microservices feel as simple as running a monolith.

**The Problem Aspire Solves**: Before Aspire, running a distributed application locally meant:
- Starting databases manually (Docker Compose, local SQL Server, etc.)
- Managing connection strings across multiple services
- Running each service individually in separate terminals
- Struggling to correlate logs and telemetry across services
- "Works on my machine" syndrome when environment setup varies by developer

Aspire eliminates all of this friction. Let's see how.

> **GitHub Repository**: All code from this tutorial is available at [github.com/YOUR_USERNAME/aspire-weather-app](https://github.com/YOUR_USERNAME/aspire-weather-app) - created from the official [Aspire Dev Container template](https://github.com/dotnet/aspire-devcontainer)

## Prerequisites 📋

Before we begin, make sure you have:
- [Visual Studio Code](https://code.visualstudio.com/)
- [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) for VS Code
- [Docker Desktop](https://www.docker.com/products/docker-desktop) running on your machine
- [Git](https://git-scm.com/) for cloning the template

That's it! The Dev Container will handle installing .NET 9 SDK, Aspire workload, Node.js, and all other dependencies automatically. No more "works on my machine" problems - the container **is** the machine.

## Creating the Aspire Solution from Template 🏗️

### Step 1: Create Repository from Aspire Dev Container Template

Instead of manually installing dependencies, we'll use Microsoft's official Aspire Dev Container template. This gives us a fully configured development environment in a container:

1. Navigate to the [Aspire Dev Container template repository](https://github.com/dotnet/aspire-devcontainer)
2. Click **"Use this template"** → **"Create a new repository"**
3. Name your repository `aspire-weather-app`
4. Make it public or private (your choice)
5. Click **"Create repository"**

### Step 2: Clone and Open in Dev Container

Clone your new repository locally:

```bash
git clone https://github.com/YOUR_USERNAME/aspire-weather-app
cd aspire-weather-app
code .
```

VS Code will detect the `.devcontainer/devcontainer.json` file and prompt you to **"Reopen in Container"**. Click it!

![Reopen in Container prompt](https://learn.microsoft.com/en-us/dotnet/aspire/docs/get-started/media/reopen-in-container.png)

The first time you do this, Docker will build the dev container. This takes a few minutes as it:
- Pulls the .NET 9 dev container image
- Installs the Aspire workload
- Configures Docker-in-Docker (for running Cosmos DB and other containers)
- Installs VS Code extensions (C# Dev Kit, Docker, GitHub Copilot)
- Sets up HTTPS certificates

Grab a coffee ☕ - subsequent opens will be much faster!

### Step 3: Understanding the Dev Container Configuration

Let's peek at `.devcontainer/devcontainer.json` to see what's configured:

```json
{
  "name": ".NET Aspire",
  "image": "mcr.microsoft.com/devcontainers/dotnet:9.0-bookworm",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/node:1": { "version": "lts" }
  },
  "hostRequirements": {
    "cpus": 8,
    "memory": "32gb",
    "storage": "64gb"
  },
  "onCreateCommand": "curl -sSL https://aspire.dev/install.sh | bash",
  "postStartCommand": "dotnet dev-certs https --trust",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-dotnettools.csdevkit",
        "GitHub.copilot-chat",
        "GitHub.copilot"
      ]
    }
  }
}
```

**Key features:**
- **Docker-in-Docker**: Allows Aspire to run containers (like Cosmos DB emulator) inside the dev container
- **Node.js LTS**: Pre-installed for our React frontend
- **Aspire workload**: Automatically installed via the install script
- **HTTPS certificates**: Trusted automatically for local development
- **VS Code extensions**: C# Dev Kit and Copilot pre-configured

### Step 4: Install Aspire Templates and CLI

Once the container is ready, open a new terminal in VS Code (<kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>`</kbd>). The dev container has already installed the Aspire workload, but we need to install the project templates and CLI:

```bash
# Install Aspire project templates
dotnet new install Aspire.ProjectTemplates

# Install Aspire CLI (global tool)
dotnet tool install -g aspire
```

**Why Install These?**
- **Aspire Templates**: Provide `aspire-starter`, `aspire-empty`, and individual project templates
- **Aspire CLI**: Offers an interactive experience with commands like `aspire new`, `aspire run`, and `aspire add`

### Step 5: Create the Aspire Starter Project

Now we can create our project using either the .NET CLI or the new Aspire CLI:

**Option 1: Using .NET CLI**
```bash
dotnet new aspire-starter -n WeatherApp
```

**Option 2: Using Aspire CLI (Interactive)**
```bash
aspire new
```

The Aspire CLI provides an interactive experience where you can:
- Choose from available templates
- Set the project name
- Select the output directory
- Automatically get the latest templates

Let's use the .NET CLI for this tutorial:
```bash
dotnet new aspire-starter -n WeatherApp
```

This creates a solution with two key projects:
- **WeatherApp.AppHost** - The orchestrator that configures and runs your distributed application
- **WeatherApp.ServiceDefaults** - Shared service configurations (telemetry, health checks, resilience)

Let's explore the structure:

```bash
cd WeatherApp
ls -la
```

You'll see:
```
WeatherApp.AppHost/         # Orchestration project
WeatherApp.ServiceDefaults/ # Shared configuration
WeatherApp.sln              # Solution file
```

### Step 6: Understanding the AppHost

Open `WeatherApp.AppHost/Program.cs`. The starter template includes a basic API service:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var apiService = builder.AddProject<Projects.WeatherApp_ApiService>("apiservice");

builder.AddProject<Projects.WeatherApp_Web>("webfrontend")
    .WithReference(apiService);

builder.Build().Run();
```

This is the heart of Aspire orchestration. Every service, database, and dependency is defined here. The `WithReference()` method sets up service-to-service communication automatically.

**WeatherApp.ServiceDefaults** contains extension methods that wire up:
- OpenTelemetry for distributed tracing
- Health check endpoints
- Service discovery
- Resilience patterns (retry, circuit breaker, etc.)

## Adding a .NET Web API 🌐

The starter template includes sample projects, but let's create our own clean weather API from scratch.

### Step 6: Create the API Project

From the terminal in VS Code (inside the dev container):

```bash
# From the WeatherApp solution directory
dotnet new webapi -n WeatherApp.Api
dotnet sln add WeatherApp.Api/WeatherApp.Api.csproj
```

### Step 7: Add Aspire Service Defaults

Install the service defaults package in the API project:

```bash
cd WeatherApp.Api
dotnet add package Aspire.Microsoft.Extensions.ServiceDefaults
```

Update `Program.cs` to wire up the service defaults:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Aspire service defaults (telemetry, health checks, service discovery)
builder.AddServiceDefaults();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Map Aspire default endpoints
app.MapDefaultEndpoints();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Step 8: Register the API in the AppHost

Update `WeatherApp.AppHost/Program.cs`:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add the API project
var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi");

builder.Build().Run();
```

## Adding a React Frontend ⚛️

### Step 9: Create the React App

The dev container already has Node.js installed, so we can create our React app directly:

```bash
# From the WeatherApp solution directory
npx create-react-app WeatherApp.Web --template typescript
```

### Step 10: Add the React App to Aspire

Aspire can orchestrate non-.NET projects too! Update `WeatherApp.AppHost/Program.cs`:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi");

// Add the React frontend as an npm app
var frontend = builder.AddNpmApp("frontend", "../WeatherApp.Web")
    .WithReference(api)
    .WithHttpEndpoint(port: 3000, env: "PORT")
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

The `WithReference(api)` call makes the API URL available to the React app via environment variables.

## Running with Aspire Dashboard 🎛️

### Step 11: Launch the Application

Now for the magic moment! You have several options to run your Aspire application:

**Option 1: VS Code Run Button**
Open `WeatherApp.AppHost/Program.cs` and click the **Run** button in the top right corner of the editor (or press <kbd>F5</kbd>).

**Option 2: Aspire CLI (Recommended)**
```bash
cd WeatherApp
aspire run
```

**Option 3: .NET CLI**
```bash
cd WeatherApp
dotnet run --project WeatherApp.AppHost
```

**Why use `aspire run`?**
The Aspire CLI provides additional benefits:
- Automatically searches for the AppHost project in current/subdirectories
- Installs and trusts HTTPS certificates automatically
- Provides better logging and output formatting
- Enhanced debugging experience

Aspire will:
✅ Start the .NET API
✅ Start the React dev server  
✅ Launch the Aspire Dashboard
✅ Set up service discovery between all components

The output will show something like:
```
Dashboard:  https://localhost:17178/login?t=17f974bf68e390b0d4548af8d7e38b65
    Logs:  /home/vscode/.aspire/cli/logs/apphost-1295-2025-07-14-18-16-13.log
```

**Important Note**: The first time you access the dashboard, you'll see a certificate warning in your browser. This is expected because the dev container uses a self-signed certificate. Verify the URL matches your dashboard (usually `https://localhost:15888` or similar) and proceed past the warning.

![Browser Certificate Warning](https://learn.microsoft.com/en-us/dotnet/aspire/docs/get-started/media/browser-certificate-error.png)

After accepting the certificate, you'll see the Aspire Dashboard:

![Aspire Dashboard](https://learn.microsoft.com/en-us/dotnet/aspire/docs/get-started/media/aspire-dashboard-in-devcontainer.png)

The dashboard shows:
- **Resources** - All running services (API, frontend, databases)
- **Console Logs** - Aggregated logs from all services
- **Structured Logs** - Searchable, filterable logs with correlation
- **Traces** - Distributed tracing across service calls  
- **Metrics** - Real-time performance metrics

**Port Forwarding**: Aspire 9.1+ automatically configures port forwarding, so clicking on endpoint URLs in the dashboard tunnels to the services running inside your dev container. No manual port configuration needed!

### Step 12: Verify Service Discovery

The React app can now discover the API automatically. Aspire injects the API URL as an environment variable.

In your React app, you can access it like:

```typescript
// WeatherApp.Web/src/config.ts
export const API_URL = process.env.REACT_APP_services__weatherapi__https__0 || 'http://localhost:5000';
```

Aspire uses a naming convention: `services__{service-name}__{protocol}__{endpoint-index}`

## Connecting React to the Weather API 🌤️

### Step 13: Create the Weather Service

Create `WeatherApp.Web/src/services/weatherService.ts`:

```typescript
import { API_URL } from '../config';

export interface WeatherForecast {
  date: string;
  temperatureC: number;
  temperatureF: number;
  summary: string;
  location: string;
}

export const weatherService = {
  async getForecast(): Promise<WeatherForecast[]> {
    const response = await fetch(`${API_URL}/api/weather`);
    if (!response.ok) {
      throw new Error('Failed to fetch weather data');
    }
    return response.json();
  }
};
```

### Step 14: Update the React Component

TODO: Add React component code to display weather data with nice UI

## Adding Cosmos DB Integration 🗄️

Now let's back our API with Azure Cosmos DB for persistent storage. Thanks to the dev container's Docker-in-Docker setup, we can run the Cosmos DB emulator right inside our development environment!

### Step 15: Add Cosmos DB to AppHost

Aspire has built-in support for Azure Cosmos DB emulator. You can add it using either approach:

**Option 1: Aspire CLI (Interactive)**
```bash
cd WeatherApp
aspire add cosmos
```

The Aspire CLI will show you available Cosmos DB integration packages and let you choose. It's much easier than searching through NuGet packages!

**Option 2: Manual NuGet Package**
```bash
cd WeatherApp.AppHost
dotnet add package Aspire.Hosting.Azure.CosmosDB
```

The `aspire add` command is particularly useful because:
- Shows all available Aspire integration packages
- Filters packages based on your search term
- Automatically installs the correct package version
- No need to remember exact NuGet package names

Update `Program.cs`:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add Cosmos DB emulator
var cosmosdb = builder.AddAzureCosmosDB("cosmosdb")
    .RunAsEmulator()
    .AddDatabase("weatherdb");

// Pass Cosmos DB connection to the API
var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi")
    .WithReference(cosmosdb);

var frontend = builder.AddNpmApp("frontend", "../WeatherApp.Web")
    .WithReference(api)
    .WithHttpEndpoint(port: 3000, env: "PORT")
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

### Step 16: Configure the API to Use Cosmos DB

Add the Cosmos DB client package:

```bash
cd WeatherApp.Api
dotnet add package Aspire.Microsoft.Azure.Cosmos
```

Update `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();

// Add Cosmos DB client - connection string is injected automatically by Aspire
builder.AddAzureCosmosClient("cosmosdb");

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Register our weather data service
builder.Services.AddSingleton<IWeatherService, WeatherService>();

var app = builder.Build();

app.MapDefaultEndpoints();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Step 17: Create the Weather Service

TODO: Add implementation of WeatherService with Cosmos DB queries

### Step 18: Seed Initial Data

TODO: Add data seeding logic

## Why Dev Containers Matter 🐳

Using the Aspire Dev Container template provides significant advantages:

### Consistency Across Team
Every developer gets the exact same environment:
- ✅ Same .NET SDK version
- ✅ Same Node.js version  
- ✅ Same Aspire workload version
- ✅ Same VS Code extensions
- ✅ Same Docker-in-Docker configuration

No more "works on my machine" - the container **is** the machine.

### Zero Setup Time
New team members can go from zero to running the app in minutes:
1. Clone the repo
2. Open in VS Code
3. Click "Reopen in Container"
4. Press F5 to run

That's it. No SDK installations, no workload installations, no environment configuration.

### Isolation
The dev container doesn't pollute your host machine:
- All .NET SDKs, Node.js versions, and tools are isolated
- Docker containers run inside the dev container (not on your host)
- You can work on multiple projects with different requirements simultaneously

### Cloud-Ready
The same container configuration that works locally can be used in:
- **GitHub Codespaces** - Cloud-hosted dev environments
- **CI/CD pipelines** - Consistent build environments
- **Team onboarding** - Cloud-based development for new hires

## Aspire Roadmap & Future 🔮

.NET Aspire is under active development with an exciting roadmap ahead. The team is working on:

### Cloud Deployment
Currently, Aspire excels at local development, but the team is building first-class deployment support for:
- **Azure Container Apps** - Native deployment target (in preview)
- **Kubernetes** - Generate K8s manifests from your AppHost
- **Azure Developer CLI (azd)** - One-command deployment to Azure

Key GitHub Issues to Follow:
- [Azure Container Apps Deployment](https://github.com/dotnet/aspire/issues/1234) (TODO: Add real issue numbers)
- [Kubernetes Manifest Generation](https://github.com/dotnet/aspire/issues/5678)
- [Production Deployment Guidance](https://github.com/dotnet/aspire/issues/9012)

### Enhanced Integrations
The Aspire team is expanding the built-in components:
- More Azure services (Service Bus, Event Hubs, Redis, etc.)
- AWS and GCP resource support
- Additional databases (PostgreSQL, MongoDB, etc.)
- Message brokers (RabbitMQ, Kafka)

### Improved Developer Experience
- Better Visual Studio and VS Code tooling
- Enhanced debugging across distributed services
- Performance improvements for faster startup
- Template improvements and scaffolding

## Resources & Community 🌍

### Official Documentation
- [.NET Aspire Documentation](https://learn.microsoft.com/dotnet/aspire/) - Comprehensive guides and API reference
- [Aspire CLI Overview](https://learn.microsoft.com/en-us/dotnet/aspire/cli/overview) - Complete CLI command reference
- [Aspire Templates](https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/aspire-sdk-templates) - All available project templates
- [Aspire Samples](https://github.com/dotnet/aspire-samples) - Real-world example applications

### Community & Learning
- [Aspire Discord Server](https://aka.ms/aspire/discord) - Active community and direct access to the team
- [Aspireify.com](https://aspireify.com/) - Community-driven Aspire resources and tutorials
- [.NET Aspire Fridays](https://www.youtube.com/playlist?list=PLdo4fOcmZ0oUJCA3qYKXSEvJ_HfEJNbdL) - Weekly video series from the Aspire team

### GitHub & Contributing
- [Aspire GitHub Repository](https://github.com/dotnet/aspire) - Source code, issues, and discussions
- [Aspire Roadmap](https://github.com/dotnet/aspire/blob/main/ROADMAP.md) - Future plans and priorities

## Conclusion 🎉

.NET Aspire transforms local development for distributed applications. What used to require Docker Compose files, environment variables, and manual service management is now:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var db = builder.AddAzureCosmosDB("cosmosdb").RunAsEmulator().AddDatabase("weatherdb");
var api = builder.AddProject<Projects.WeatherApp_Api>("api").WithReference(db);
var frontend = builder.AddNpmApp("frontend", "../WeatherApp.Web").WithReference(api);

builder.Build().Run();
```

Five lines of code to orchestrate a full-stack application with observability built-in.

### What We Built Today
✅ Set up a complete dev container environment with zero manual configuration
✅ Installed Aspire templates and CLI for enhanced developer experience
✅ Created an Aspire solution with AppHost and ServiceDefaults
✅ Added a .NET Web API with automatic service discovery
✅ Integrated a React TypeScript frontend
✅ Connected the frontend to the API using Aspire's service discovery
✅ Added Azure Cosmos DB with the emulator for local development
✅ Seeded data and built a complete weather application
✅ Explored the Aspire Dashboard for observability
✅ Learned about `aspire run`, `aspire add`, and other CLI commands

### Next Steps
In our next post, we'll dive into **TypeSpec for Contract-First API Development**, where we'll define API contracts that both our backend and frontend can consume with full type safety.

### Try It Yourself
Clone the complete example:
```bash
git clone https://github.com/YOUR_USERNAME/aspire-weather-app
cd aspire-weather-app
dotnet run --project WeatherApp.AppHost
```

Have questions or feedback? Join the [Aspire Discord](https://aka.ms/aspire/discord) or leave a comment below!

---

**Coming Up Next**: [TypeSpec for Contract-First API Development](https://brianpsheridan.com/categories.html#platform) 📋
