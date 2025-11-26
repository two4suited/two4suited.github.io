---
title: "Building a Modern Development Platform: Service Virtualization with WireMock 🎭"
date: 2025-11-25T06:00:00-07:00
draft: false
categories: ["platform","wiremock","mocking","testing","service-virtualization"]
description: "Implementing service virtualization strategies with WireMock and WireMock Cloud for local development, integration testing, and reducing infrastructure costs"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## 🎯 Overview

In modern application development, you rarely control all the systems your application depends on. Third-party APIs, legacy services, payment gateways, and other downstream dependencies are often out of your control. Testing against these systems presents challenges: they may be unavailable, slow, expensive to call repeatedly, or unpredictable in their responses.

**Service virtualization** solves this problem by creating realistic mock versions of external services that you can run locally or in the cloud. This post explores how **WireMock** and **WireMock Cloud** enable you to:

- ✨ Mock external APIs with realistic responses
- 🚀 Develop and test locally without calling production services
- 💡 Simulate error conditions and edge cases
- 🔄 Integrate seamlessly with .NET Aspire for development environments
- 💰 Reduce infrastructure costs and API call charges

## 📚 Why Service Virtualization Matters

### The Challenge

When building modern distributed applications, you depend on many external services:

```
Your App
  ├── Payment Gateway (Stripe, Square)
  ├── Email Service (SendGrid, SES)
  ├── Third-party Data APIs
  ├── Legacy .NET Framework Services
  └── Cloud Services (Azure, AWS, GCP)
```

Testing against these systems directly is problematic:

- **Availability**: External services may have rate limits, scheduled downtime, or authentication requirements
- **Cost**: Each API call costs money; repeated test runs add up quickly
- **Consistency**: External services may have varying response times or introduce network latency
- **Testing Edge Cases**: Simulating errors, timeouts, and rate limits from real services is difficult
- **Team Coordination**: Local development requires access to shared test accounts and credentials

### The Solution

Service virtualization creates lightweight, realistic mock services that behave like their real counterparts. You record actual API responses and replay them consistently.

## 🏗️ Understanding WireMock

**WireMock** is an open-source service virtualization tool that intercepts HTTP requests and returns predefined responses. It's powerful, flexible, and free.

### WireMock Ecosystem

| Tool | Use Case |
|------|----------|
| **WireMock (OSS)** | Self-hosted mocking, local development, open-source |
| **WireMock Cloud** | Managed cloud mocking, team collaboration, hosting |
| **WireMock Docker** | Containerized mocks, local testing environments |
| **WireMock CLI** | Command-line tool for local mock management and testing |

## 🔧 Installing WireMock CLI

The **WireMock CLI** is a command-line tool that makes it easy to create, manage, and test mocks locally without Docker. It's particularly useful for quick local testing and scripting mock creation.

### Installation

Install the WireMock CLI globally using npm:

```bash
npm install --global @wiremock/cli
```

Verify the installation:

```bash
wiremock --version
```

### 🚀 Starting a Local WireMock Server

Once installed, you can start a WireMock server with a single command:

```bash
# Start WireMock on default port 8080
wiremock

# Start on a custom port
wiremock --port 9090

# Start with verbose logging
wiremock --verbose
```

The server will be available at `http://localhost:8080`.

### 📋 Creating Mocks via CLI

Create a `mocks.json` file in your project:

```json
{
  "mappings": [
    {
      "id": "weather-001",
      "request": {
        "method": "GET",
        "url": "/api/weather/forecasts"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "application/json"
        },
        "jsonBody": [
          {
            "id": "550e8400-e29b-41d4-a716-446655440001",
            "date": "2025-11-23",
            "temperatureC": 22,
            "temperatureF": 72,
            "summary": "Sunny",
            "location": "New York, USA"
          }
        ]
      }
    }
  ]
}
```

Start WireMock with your mock definitions:

```bash
wiremock --load-mappings-from mocks.json
```

### 🧪 Testing Your Mocks

Once the server is running, test your mock endpoints:

```bash
# Test the weather forecast endpoint
curl http://localhost:8080/api/weather/forecasts

# You should see the array of weather data
```

### 🔄 Comparison: CLI vs. Docker vs. Cloud

| Feature | CLI | Docker | Cloud |
|---------|-----|--------|-------|
| **Installation** | npm install | Docker required | Browser only |
| **Startup Time** | ~1 second | ~5-10 seconds | Instant |
| **File Persistence** | Local filesystem | Volume mount | Cloud storage |
| **Team Collaboration** | Version control repo | Version control repo | Web UI |
| **Portability** | Node.js required | Docker engine required | Network access only |
| **Best For** | Quick local testing | CI/CD pipelines | Team development |

### 💡 VS Code Integration with MCP

The WireMock CLI can also be configured as a **Model Context Protocol (MCP) server** in VS Code, giving you Copilot access to WireMock capabilities directly in your editor.

#### Configure WireMock MCP Server

Add this to your VS Code `settings.json`:

```json
{
  "mcp": {
    "servers": {
      "WireMock": {
        "type": "stdio",
        "command": "wiremock",
        "args": ["mcp"]
      }
    }
  }
}
```

**How to access VS Code settings:**
1. Open VS Code settings: `Cmd + ,` (macOS) or `Ctrl + ,` (Windows/Linux)
2. Click the **{}** icon in the top-right to open `settings.json`
3. Add or merge the `mcp` configuration above
4. Save and restart VS Code

#### Using WireMock in VS Code with Copilot

Once configured, you can use Copilot to:

- 🎯 Generate mock definitions directly in your editor
- 📝 Create stub mappings from existing API responses
- 🧪 Test mocks and verify responses without leaving VS Code
- 🔄 Manage mock configurations through natural language

**Example interactions:**

```
You: "Create a mock for a weather API endpoint that returns forecast data"
Copilot: [Generates mock definition]

You: "Show me how to add error scenarios to my weather mocks"
Copilot: [Provides examples with 4XX and 5XX responses]
```

This integration brings WireMock's power directly into your development workflow, making it faster to create and iterate on mocks.

## 🌍 Getting Started with WireMock Cloud

WireMock Cloud provides a managed service for creating, hosting, and managing mocks without running your own infrastructure.

### 📋 Create Your First Mock in WireMock Cloud

1. **Sign Up**: Visit [https://www.wiremock.io](https://www.wiremock.io) and create an account
2. **Authenticate**: Run `wiremock login` to connect your CLI to WireMock Cloud
3. **Create a New Mock API**: Name it something descriptive like "Weather API Mock"
4. **Define Your Stub**: Set up a request/response pair

#### Example: Mocking a Weather API

Let's create a practical mock for a weather forecast API that returns data for multiple cities:

**Creating the Mock with WireMock CLI**:

```bash
# Login to WireMock Cloud (if not already logged in)
wiremock login

# Create a new mock API
wiremock create-mock-api "Weather API Mock" --hostname weather-api-mock
```

**Request Matcher** (GET all forecasts):
```json
{
  "method": "GET",
  "url": "/api/weather/forecasts"
}
```

**Response** (Array of weather forecasts):
```json
{
  "status": 200,
  "headers": {
    "Content-Type": ["application/json"]
  },
  "body": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "date": "2025-11-23",
      "temperatureC": 22,
      "temperatureF": 72,
      "summary": "Sunny",
      "location": "New York, USA"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "date": "2025-11-24",
      "temperatureC": 18,
      "temperatureF": 64,
      "summary": "Partly Cloudy",
      "location": "London, UK"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "date": "2025-11-25",
      "temperatureC": 5,
      "temperatureF": 41,
      "summary": "Rainy",
      "location": "Dublin, Ireland"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440004",
      "date": "2025-11-26",
      "temperatureC": 28,
      "temperatureF": 82,
      "summary": "Clear",
      "location": "Sydney, Australia"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440005",
      "date": "2025-11-27",
      "temperatureC": -5,
      "temperatureF": 23,
      "summary": "Snowy",
      "location": "Toronto, Canada"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440006",
      "date": "2025-11-28",
      "temperatureC": 35,
      "temperatureF": 95,
      "summary": "Hot",
      "location": "Phoenix, USA"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440007",
      "date": "2025-11-29",
      "temperatureC": 12,
      "temperatureF": 54,
      "summary": "Cloudy",
      "location": "Paris, France"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440008",
      "date": "2025-11-30",
      "temperatureC": 15,
      "temperatureF": 59,
      "summary": "Drizzle",
      "location": "Amsterdam, Netherlands"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440009",
      "date": "2025-12-01",
      "temperatureC": 8,
      "temperatureF": 46,
      "summary": "Overcast",
      "location": "Berlin, Germany"
    },
    {
      "id": "550e8400-e29b-41d4-a716-446655440010",
      "date": "2025-12-02",
      "temperatureC": 32,
      "temperatureF": 90,
      "summary": "Humid",
      "location": "Tokyo, Japan"
    }
  ]
}
```

**Testing the Mock**:

```bash
# Get all weather forecasts
curl https://weather-api-mock.eu.wiremock.io/api/weather/forecasts

# You should see the full array of 10 weather forecasts
```

### 🔐 Security Considerations

When using WireMock Cloud:

- ✅ Keep your mock URLs private (treat them like API keys)
- ✅ Use authentication headers in requests
- ✅ Enable HTTPS for all communication
- ✅ Limit mock APIs to development environments only
- ⚠️ Never use production data in mock responses

## 🏗️ Integrating WireMock Cloud with .NET Aspire

.NET Aspire is perfect for local development because it manages your application's infrastructure. You can wire WireMock Cloud mocks directly into your Aspire orchestration.

### Setting Up the Aspire Project

First, create your mock in WireMock Cloud and get its public URL:

```
https://your-instance.eu.wiremock.io
```

### 📋 Configuring HttpClient in Aspire

In your `Program.cs`, add your mock endpoint:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Reference your WireMock Cloud instance
var mockWeatherService = builder
    .AddConnectionString("WeatherServiceUrl", 
        "https://weather-api-mock.eu.wiremock.io");

builder
    .AddProject<Projects.YourApp>("yourapp")
    .WithEnvironment("WeatherService__BaseUrl", 
        mockWeatherService);

await builder.Build().RunAsync();
```

### Using the Mock in Your Application

In your application's `appsettings.Development.json`:

```json
{
  "WeatherService": {
    "BaseUrl": "https://weather-api-mock.eu.wiremock.io"
  }
}
```

In your service class:

```csharp
public class WeatherService
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl;

    public WeatherService(HttpClient httpClient, IConfiguration config)
    {
        _httpClient = httpClient;
        _baseUrl = config["WeatherService:BaseUrl"];
    }

    public async Task<List<WeatherForecast>> GetForecastsAsync()
    {
        var response = await _httpClient.GetAsync(
            $"{_baseUrl}/api/weather/forecasts");
        
        return await response.Content.ReadAsAsync<List<WeatherForecast>>();
    }
}
```

Register in dependency injection:

```csharp
builder.Services.AddHttpClient<WeatherService>()
    .ConfigureHttpClient(client => 
    {
        var config = builder.Configuration;
        client.BaseAddress = new Uri(
            config["WeatherService:BaseUrl"] ?? "");
    });
```

## 📥 Downloading Mocks for Local Development

While WireMock Cloud is convenient for early development and team collaboration, you may want to run mocks locally. WireMock allows you to export your mock definitions and run them in a Docker container.

### 📋 Export Mocks from WireMock Cloud

1. In WireMock Cloud UI, navigate to your mock API
2. Go to **Settings** → **Export**
3. Download the mock definition (typically a JSON file or Swagger spec)

### 🐳 Running WireMock Locally with Docker

Create a `docker-compose.yml` for your development environment:

```yaml
version: '3.8'

services:
  weather-mock:
    image: wiremock/wiremock:latest
    ports:
      - "8080:8080"
    volumes:
      - ./mocks:/home/wiremock/mappings
      - ./responses:/home/wiremock/__files
    environment:
      - WIREMOCK_ENABLE_STUBBING=true
      - WIREMOCK_VERBOSE=true
    command: 
      - --port=8080
      - --disable-request-logging=false
```

### 📋 Structure Your Mock Files

WireMock uses a specific directory structure:

```
project-root/
├── docker-compose.yml
└── mocks/
    └── mappings/
        └── weather-api.json
```

#### Mock Definition Example (`mocks/mappings/weather-api.json`):

```json
{
  "id": "weather-001",
  "request": {
    "method": "GET",
    "url": "/api/weather/forecasts"
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "date": "2025-11-23",
        "temperatureC": 22,
        "temperatureF": 72,
        "summary": "Sunny",
        "location": "New York, USA"
      }
    ]
  }
}
```

### 🚀 Running the Local Container

```bash
# Start the mock service
docker-compose up -d weather-mock

# Verify it's running
curl http://localhost:8080/api/weather/forecasts

# View logs
docker-compose logs -f weather-mock

# Stop the service
docker-compose down
```

## 🔄 Comparing Cloud vs. Local Mocks

| Aspect | WireMock Cloud | Local Container |
|--------|----------------|-----------------|
| **Setup** | Browser UI, no infrastructure | Docker Compose, local config |
| **Collaboration** | ✅ Team-friendly | ❌ Requires repository sync |
| **Performance** | Network latency | Local latency |
| **Cost** | Free tier, paid plans | Free (self-hosted) |
| **Customization** | Limited UI | Full control via JSON |
| **Persistence** | Cloud-managed | Container ephemeral |
| **CI/CD Integration** | Easy API integration | Docker registry ready |

### When to Use Each

**Use WireMock Cloud when:**
- 👥 Multiple team members need access to mocks
- 🏃 Rapid prototyping without infrastructure setup
- 🔗 Testing integration with external systems
- 📊 You want centralized mock management

**Use Local Docker Container when:**
- 🚀 Deploying mocks to CI/CD pipelines
- 🔧 Fine-grained control over mock behavior
- 💻 Working offline or with unstable internet
- 🎯 Local development with reproducible environments

## 🧪 Advanced Mocking Scenarios

### Simulating Errors and Edge Cases

One of mocking's greatest strengths is testing error scenarios safely.

#### 401 Unauthorized Response

```json
{
  "id": "weather-401",
  "priority": 1,
  "request": {
    "method": "GET",
    "url": "/api/weather/forecasts",
    "headers": {
      "Authorization": {
        "matches": "Bearer invalid.*"
      }
    }
  },
  "response": {
    "status": 401,
    "headers": {
      "Content-Type": "application/json"
    },
    "jsonBody": {
      "error": "Unauthorized",
      "message": "Invalid API key"
    }
  }
}
```

#### Simulating Timeout/Delay

```json
{
  "id": "weather-timeout",
  "request": {
    "method": "GET",
    "url": "/api/weather/forecasts"
  },
  "response": {
    "status": 200,
    "jsonBody": [{
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "date": "2025-11-23",
      "temperatureC": 22,
      "summary": "Sunny"
    }],
    "delayDistribution": {
      "type": "lognormal",
      "median": 3000,
      "sigma": 0.5
    }
  }
}
```

#### 5XX Server Error

```json
{
  "id": "weather-error",
  "priority": 2,
  "request": {
    "method": "GET",
    "url": "/api/weather/forecasts",
    "queryParameters": {
      "simulate_error": {
        "equalTo": "true"
      }
    }
  },
  "response": {
    "status": 500,
    "jsonBody": {
      "error": "Internal Server Error",
      "message": "Weather service temporarily unavailable"
    }
  }
}
```

### Request Matching with Priorities

WireMock matches requests in priority order (higher number = higher priority). This lets you create catch-all patterns with specific overrides:

```json
[
  {
    "id": "specific-location",
    "priority": 10,
    "request": {
      "method": "GET",
      "url": "/api/weather/forecasts",
      "queryParameters": {
        "location": {
          "equalTo": "New York"
        }
      }
    },
    "response": {
      "status": 200,
      "jsonBody": [{
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "date": "2025-11-23",
        "temperatureC": 22,
        "summary": "Sunny",
        "location": "New York, USA"
      }]
    }
  },
  {
    "id": "default-case",
    "priority": 1,
    "request": { "method": "GET", "url": "/api/weather/forecasts" },
    "response": {
      "status": 200,
      "jsonBody": []
    }
  }
]
```

## 🔧 Aspire with Local WireMock Container

You can also integrate the local Docker container with Aspire:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add WireMock container to Aspire
var mockWeather = builder
    .AddContainer("weather-mock", "wiremock/wiremock")
    .WithEndpoint(containerPort: 8080, hostPort: 8080)
    .WithVolume("./mocks", "/home/wiremock/mappings");

builder
    .AddProject<Projects.YourApp>("yourapp")
    .WithEnvironment("WeatherService__BaseUrl", 
        "http://localhost:8080")
    .WaitFor(mockWeather);

await builder.Build().RunAsync();
```

This ensures the mock service starts before your application and provides a fully integrated local development experience.

## ✅ Best Practices for Service Virtualization

1. **Keep Mocks in Version Control**: Store mock definitions in your repository alongside your code
2. **Document Mock Behavior**: Comment your mock definitions explaining what scenarios they cover
3. **Regular Updates**: When external APIs change, update your mocks accordingly
4. **Test Error Cases**: Always include mocks for error scenarios (4XX, 5XX responses)
5. **Use Realistic Data**: Mock responses should match the real API's structure and data patterns
6. **Separate by Environment**: Have different mock sets for development, integration testing, and staging
7. **Monitor Mock Usage**: Track which mocks are being used to identify stale definitions
8. **Version Your Mocks**: If an external API changes, version your mock definitions

## 🚀 Next Steps

Now that you understand service virtualization with WireMock, you can:

- 🌍 Create mocks for all your external dependencies
- 🔄 Integrate them into your Aspire development environment
- 🧪 Test error scenarios and edge cases confidently
- 📊 Reduce infrastructure costs and external API calls during development
- 👥 Enable faster, more reliable local development for your team

In the next post, we'll explore how to integrate these mocking strategies into your CI/CD pipelines with Azure DevOps, ensuring your integration tests are consistent and fast.

