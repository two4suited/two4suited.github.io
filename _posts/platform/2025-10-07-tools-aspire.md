---
title: "Building a Modern Development Platform:2. Name your repository `blog-platform-aspire`
3. Make it public or private (your choice)
4. Click **"Create repository"**

### Step 2: Clone and Open in Dev Container

Clone your n### Step 7: Add Aspire Service Defaults

Add a project reference to the ServiceDefaults project in the API project:

```bash
cd src/WeatherApp.Api
dotnet add reference ../WeatherApp.ServiceDefaults/WeatherApp.ServiceDefaults.csproj
```itory locally:

```bash
git clone https://github.com/two4suited/blog-platform-aspire
cd blog-platform-aspiree for Local Development 💻"
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

> **GitHub Repository**: All code from this tutorial is available at [github.com/two4suited/blog-platform-aspire](https://github.com/two4suited/blog-platform-aspire) - created from the official [Aspire Dev Container template](https://github.com/dotnet/aspire-devcontainer)

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

### Step 3: Understanding the Dev Container Setup

The Aspire dev container template automatically configures everything you need for Aspire development:

**Everything is Pre-Installed!** 🎉
- .NET 9 SDK ✅
- Aspire workload ✅  
- Aspire project templates ✅
- Aspire CLI ✅
- HTTPS certificates ✅

> **For complete dev container configuration details** (including Node.js and Docker setup for full-stack applications), see the official [Aspire Dev Containers documentation](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/dev-containers).

### Step 4: Create Aspire Projects Using CLI Template Selection

We'll use the Aspire CLI's interactive experience to create our projects:

```bash
aspire new
```

The Aspire CLI will guide you through the setup:

1. **Template Selection Menu:**
   ```
   Select a project template:
   > Starter template
     AppHost and service defaults
     AppHost  
     Service defaults
     Integration tests
   ```

2. **Select "AppHost and service defaults"** - this creates both projects we need in one step

3. **Project Name:** When prompted, enter `WeatherApp`

4. **Output Location:** When prompted for the output directory, enter `src` (this will create the `src` directory for you)

This creates both essential components:
- **AppHost** - The orchestrator that manages all our services
- **Service defaults** - Shared configurations for telemetry, health checks, and resilience

This will create in the `src/` directory:
- `WeatherApp.AppHost/` - Orchestration project
- `WeatherApp.ServiceDefaults/` - Shared configuration project  
- `WeatherApp.sln` - Solution file tying them together

**Why "AppHost and service defaults"?**
This template gives us:
- Clean foundation without sample projects (unlike Starter template)
- Both essential Aspire components in one step
- Proper project references already configured
- Ready to add our own API and frontend projects

Let's explore the structure that was created:

```bash
cd src
ls -la
```

You'll see:
```
WeatherApp.AppHost/         # Orchestration project
WeatherApp.ServiceDefaults/ # Shared configuration
WeatherApp.sln              # Solution file
```

### Step 5: Understanding the AppHost

Open `src/WeatherApp.AppHost/AppHost.cs`. The AppHost template has a minimal setup:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add your resources here

builder.Build().Run();
```

This is the heart of Aspire orchestration. Every service, database, and dependency will be defined here. As we add projects, we'll use methods like:
- `builder.AddProject<>()` to add .NET projects
- `builder.AddNpmApp()` to add Node.js/React apps
- `builder.AddAzureCosmosDB()` to add databases
- `.WithReference()` to set up service-to-service communication

**WeatherApp.ServiceDefaults** contains extension methods that wire up:
- OpenTelemetry for distributed tracing
- Health check endpoints
- Service discovery
- Resilience patterns (retry, circuit breaker, etc.)

## Running the Basic Aspire Dashboard 🎛️

Before we add any services, let's see Aspire in action with just the basic setup:

```bash
aspire run
```

**What `aspire run` does:**
- Automatically searches for and finds the AppHost project in the current directory and subdirectories
- Creates configuration in the `.aspire` folder with `settings.json` for project paths
- Builds the AppHost project
- Starts the Aspire Dashboard (even with no services yet)
- Shows the foundation for our distributed application

The output will show something like:
```
Dashboard:  https://localhost:17178/login?t=17f974bf68e390b0d4548af8d7e38b65
    Logs:  /home/vscode/.aspire/cli/logs/apphost-1295-2025-07-14-18-16-13.log
```

**Certificate Warning**: The first time you access the dashboard URL, you'll see a certificate error in your browser. This is expected behavior in the dev container environment. 

> **For complete details on handling certificate warnings**, see the [Aspire Dev Containers documentation](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/dev-containers#quick-start-using-template-repository).

Once you access the dashboard, you'll see it's ready and waiting for services! The dashboard shows:
- **Resources** - Currently empty, but ready for our services
- **Console Logs** - Dashboard and AppHost logs
- **Structured Logs** - Searchable logs with correlation IDs
- **Traces** - Ready for distributed tracing
- **Metrics** - Performance monitoring ready

**Stop the Application**: Press <kbd>Ctrl</kbd>+<kbd>C</kbd> in the terminal to stop Aspire when you're ready to continue.

Now let's add some services to see the real power of Aspire orchestration!

## Adding a .NET Web API 🌐

The starter template includes sample projects, but let's create our own clean weather API from scratch.

### Step 6: Create the API Project

From the terminal in VS Code (inside the dev container):

```bash
# Navigate to the src directory where our projects are created
cd src
dotnet new webapi -n WeatherApp.Api
dotnet sln WeatherApp.sln add WeatherApp.Api/WeatherApp.Api.csproj
```

### Step 7: Add Aspire Service Defaults

Add a project reference to the ServiceDefaults project in the API project:

```bash
cd WeatherApp.Api
dotnet add reference ../WeatherApp.ServiceDefaults/WeatherApp.ServiceDefaults.csproj
```

Update `WeatherApp.Api/Program.cs` to wire up the service defaults:

**Add ServiceDefaults** after `WebApplication.CreateBuilder(args)`:
```csharp
builder.AddServiceDefaults();
```

This integrates your API with Aspire's telemetry, health checks, service discovery, and resilience patterns from the ServiceDefaults project.

### Step 8: Register the API in the AppHost

First, add a project reference from AppHost to the API:

```bash
dotnet add src/WeatherApp.AppHost/WeatherApp.AppHost.csproj reference src/WeatherApp.Api/WeatherApp.Api.csproj
```

Now update `WeatherApp.AppHost/Program.cs`:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add the API project
var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi");

builder.Build().Run();
```

## Adding a React Frontend ⚛️

### Step 9: Create the React App

The dev container already has Node.js installed, so we can create our React app with Vite:

```bash
# From the src directory  
npm create vite@latest WeatherApp.Web -- --template react-ts
cd WeatherApp.Web
npm install
```

### Step 10: Add the React App to Aspire

First, install the Aspire Community Toolkit for Node.js extensions:

```bash
dotnet add src/WeatherApp.AppHost/WeatherApp.AppHost.csproj package CommunityToolkit.Aspire.Hosting.NodeJS.Extensions
```

Now update `WeatherApp.AppHost/Program.cs` to use the Vite integration:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi");

// Add the React frontend using Vite integration
var frontend = builder.AddViteApp("frontend", "../WeatherApp.Web")
    .WithNpmPackageInstallation()
    .WithReference(api)
    .WithEnvironment("BROWSER", "false")
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

Key features of this integration:
- **`AddViteApp()`** - Uses the official Vite integration from Aspire Community Toolkit
- **`WithNpmPackageInstallation()`** - Automatically runs `npm install` before starting the dev server
- **`WithReference(api)`** - Makes the API URL available to the React app via environment variables
- **`WithExternalHttpEndpoints()`** - Exposes the Vite dev server for external access

## Running with Services Added 🚀

### Step 11: Launch the Complete Application

Now let's run our application with all services configured:

```bash
aspire run
```

This time, Aspire will:
✅ Start the .NET Weather API
✅ Start the React dev server  
✅ Launch the Aspire Dashboard
✅ Set up service discovery between all components

The dashboard now shows all your services running and their interconnections!

## Connecting React to the Weather API 🌤️

### Step 12: Configure Vite Proxy

First, configure Vite to proxy API calls to the backend. Update `WeatherApp.Web/vite.config.ts`:

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    host: true,
    proxy: {
      '/api': {
        target: process.env.services__weatherapi__https__0 || 
                process.env.services__weatherapi__http__0,
        changeOrigin: true,
        secure: false,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
})
```

This configuration:
- **`proxy: '/api'`** - Intercepts all requests starting with `/api`
- **`target`** - Uses Aspire's injected environment variables for the API URL
- **`changeOrigin: true`** - Changes the origin of the request to match the target
- **`secure: false`** - Allows self-signed certificates in development
- **`rewrite`** - Removes `/api` prefix before forwarding to the backend

### Step 13: Create the Weather Service

Create `WeatherApp.Web/src/services/weatherService.ts`:

```typescript
export interface WeatherForecast {
  date: string;
  temperatureC: number;
  temperatureF: number;
  summary: string;
  location: string;
}

export const weatherService = {
  async getForecast(): Promise<WeatherForecast[]> {
    // Use the Vite proxy - requests to /api/* get forwarded to the backend
    const response = await fetch('/api/weather');
    if (!response.ok) {
      throw new Error('Failed to fetch weather data');
    }
    return response.json();
  }
};
```

### Step 14: Update the React Component

Update `WeatherApp.Web/src/App.tsx` to display the weather data:

```typescript
import { useState, useEffect } from 'react'
import { weatherService } from './services/weatherService'
import type { WeatherForecast } from './services/weatherService'
import './App.css'

function App() {
  const [weatherData, setWeatherData] = useState<WeatherForecast[]>([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchWeatherData = async () => {
      try {
        setLoading(true)
        setError(null)
        const data = await weatherService.getForecast()
        setWeatherData(data)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Failed to fetch weather data')
      } finally {
        setLoading(false)
      }
    }

    fetchWeatherData()
  }, [])

  if (loading) {
    return <div className="loading">Loading weather data...</div>
  }

  if (error) {
    return (
      <div className="error">
        <h2>Error</h2>
        <p>{error}</p>
        <button onClick={() => window.location.reload()}>Retry</button>
      </div>
    )
  }

  return (
    <div className="weather-app">
      <h1>Weather Forecast</h1>
      <div className="weather-table-container">
        <table className="weather-table">
          <thead>
            <tr>
              <th>Date</th>
              <th>Location</th>
              <th>Temperature (°C)</th>  
              <th>Temperature (°F)</th>
              <th>Summary</th>
            </tr>
          </thead>
          <tbody>
            {weatherData.map((forecast, index) => (
              <tr key={index}>
                <td>{new Date(forecast.date).toLocaleDateString()}</td>
                <td>{forecast.location}</td>
                <td>{forecast.temperatureC}°C</td>
                <td>{forecast.temperatureF}°F</td>
                <td>{forecast.summary}</td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
    </div>
  )
}

export default App
```

This component includes:
- **Loading states** - Shows loading spinner while fetching data
- **Error handling** - Displays errors with retry option
- **TypeScript types** - Fully typed with the WeatherForecast interface
- **Clean table layout** - Displays all weather data in an organized table
- **Date formatting** - Properly formats dates for display

Add the corresponding styles in `WeatherApp.Web/src/App.css`:

```css
#root {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
}

.weather-app {
  text-align: center;
}

.weather-app h1 {
  color: #213547;
  margin-bottom: 2rem;
}

.weather-table-container {
  overflow-x: auto;
  margin: 2rem 0;
}

.weather-table {
  width: 100%;
  border-collapse: collapse;
  margin: 0 auto;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  overflow: hidden;
}

.weather-table th {
  background: #646cff;
  color: white;
  padding: 1rem;
  text-align: left;
  font-weight: 600;
}

.weather-table td {
  padding: 1rem;
  border-bottom: 1px solid #e5e5e5;
  text-align: left;
}

.weather-table tbody tr:hover {
  background: #f8f9fa;
}

.weather-table tbody tr:last-child td {
  border-bottom: none;
}

.loading {
  text-align: center;
  padding: 2rem;
  font-size: 1.2rem;
  color: #646cff;
}

.error {
  text-align: center;
  padding: 2rem;
  color: #d32f2f;
}

.error h2 {
  margin-bottom: 1rem;
}

.error button {
  background: #646cff;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  font-size: 1rem;
  margin-top: 1rem;
}

.error button:hover {
  background: #535bf2;
}
```

The CSS provides:
- **Modern styling** - Clean, professional appearance with Vite's default color scheme
- **Responsive design** - Table scrolls horizontally on smaller screens
- **Interactive elements** - Hover effects and styled buttons
- **Accessible design** - Good contrast ratios and readable typography
- **Loading/Error states** - Styled feedback for different application states

## Adding Cosmos DB Integration 🗄️

Now let's back our API with Azure Cosmos DB for persistent storage. Thanks to the dev container's Docker-in-Docker setup, we can run the Cosmos DB emulator right inside our development environment!

### Step 15: Add Cosmos DB to AppHost

Aspire has built-in support for Azure Cosmos DB emulator. You can add it using either approach:

**Option 1: Aspire CLI (Interactive)**
```bash
aspire add cosmos
```

The Aspire CLI will show you available Cosmos DB integration packages and let you choose. It's much easier than searching through NuGet packages!

**Option 2: Manual NuGet Package**
```bash
cd src/WeatherApp.AppHost
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
cd src/WeatherApp.Api
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
git clone https://github.com/two4suited/blog-platform-aspire
cd blog-platform-aspire
aspire run
```

Have questions or feedback? Join the [Aspire Discord](https://aka.ms/aspire/discord) or leave a comment below!

---

**Coming Up Next**: [TypeSpec for Contract-First API Development](https://brianpsheridan.com/categories.html#platform) 📋
