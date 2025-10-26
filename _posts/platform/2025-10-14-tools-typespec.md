---
title: "Building a Modern Development Platform: TypeSpec for Contract-First API Development 📋"
date: 2025-10-14T06:00:00-07:00
draft: false
categories: ["platform","typespec","api","contract-first","openapi"]
description: "Deep dive into using TypeSpec for contract-first API development - defining clean API contracts that generate OpenAPI specs and enable parallel team development"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

## Introduction 🚀

In our [Aspire local development post](https://brianpsheridan.com/platform/aspire/dotnet/local-development/orchestration/2025/10/07/tools-aspire.html), we built a weather application with a .NET API and React frontend. While the application works perfectly, we noticed something: the frontend and backend teams had to coordinate manually on the API contract. The React TypeScript interfaces had to match the C# models exactly, and any changes required careful synchronization.

This is where **TypeSpec** shines. Instead of defining APIs in code and hoping both teams stay in sync, TypeSpec lets us define the contract first, then generate both the OpenAPI specification and the client/server code from a single source of truth.

## What is TypeSpec? 🤔

TypeSpec is Microsoft's modern API description language designed to overcome the limitations of writing OpenAPI specifications directly. Think of it as "TypeScript for API contracts" - it provides:

**A Clean, Readable Syntax**
```typespec
model WeatherForecast {
  id: string;
  date: plainDate;
  temperatureC: int32;
  temperatureF: int32;
  summary: string;
  location: string;
}

@route("/weather")
namespace WeatherAPI {
  @get
  op getForecast(): WeatherForecast[];
}
```

Instead of this verbose OpenAPI YAML:
```yaml
paths:
  /weather:
    get:
      operationId: getForecast
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/WeatherForecast'
components:
  schemas:
    WeatherForecast:
      type: object
      properties:
        id:
          type: string
        date:
          type: string
          format: date
        temperatureC:
          type: integer
          format: int32
        # ... and so on
```

**Key Characteristics:**
- **Type-Safe**: Strong typing prevents common API contract errors
- **Composable**: Reusable models, mixins, and decorators
- **Extensible**: Plugin system for custom behaviors
- **Multi-Target**: Generate OpenAPI, JSON Schema, client SDKs, and more
- **Tooling**: Rich VS Code extension with IntelliSense and validation

## Why Would You Use TypeSpec? 💡

### 1. **Contract-First Development** 📋

Traditional development often follows this painful pattern:
1. Backend team implements API endpoints
2. Frontend team reverse-engineers the contract from API responses
3. Changes break both teams
4. Manual coordination and documentation drift

With TypeSpec:
1. Define the contract in TypeSpec
2. Generate OpenAPI spec automatically
3. Generate client SDKs for frontend
4. Generate server scaffolding for backend
5. Both teams work from the same source of truth

### 2. **Superior Developer Experience** 👨‍💻

**Type Safety Everywhere**
- Compile-time validation of your API contracts
- IntelliSense and autocomplete in VS Code
- Catch breaking changes before they reach production

**Clean, Maintainable Specifications**
- No more hand-editing massive OpenAPI YAML files
- Reusable components and mixins
- Version control friendly (readable diffs)

**Rapid Prototyping**
- Define APIs quickly without implementation overhead
- Generate mock servers for frontend development
- Validate designs before committing to implementation

### 3. **Multi-Platform Code Generation** 🏗️

From one TypeSpec definition, generate:
- **OpenAPI 3.0/3.1 specifications**
- **Client SDKs** (TypeScript, C#, Python, Java, Go)
- **Server stubs** (.NET, Node.js, Java)
- **Documentation** (interactive API docs)
- **Mock servers** for testing and development

### 4. **Enterprise-Grade Features** 🏢

**Versioning Support**
```typespec
@versioned(Versions)
namespace WeatherAPI {
  enum Versions {
    v1: "1.0",
    v2: "2.0",
  }
}
```

**Authentication & Security**
```typespec
@useAuth(BearerAuth)
@route("/weather")
op getForecast(): WeatherForecast[];
```

**Advanced Validation**
```typespec
model CreateWeatherRequest {
  @minLength(1)
  @maxLength(100)
  location: string;
  
  @minimum(-50)
  @maximum(60)
  temperatureC: int32;
}
```

### 5. **Real-World Problem Solving** 🛠️

**Problem**: Your weather API needs to support both Celsius and Fahrenheit, with validation
**TypeSpec Solution**:
```typespec
enum TemperatureUnit {
  Celsius: "C",
  Fahrenheit: "F",
}

model TemperatureReading {
  @minimum(-459.67) // Absolute zero in Fahrenheit
  value: float32;
  unit: TemperatureUnit;
}
```

**Problem**: Different clients need different response formats
**TypeSpec Solution**:
```typespec
@discriminator("kind")
union WeatherResponse {
  detailed: DetailedWeather,
  summary: WeatherSummary,
}
```

**Problem**: API needs to evolve without breaking existing clients
**TypeSpec Solution**:
```typespec
@added(Versions.v2)
model WeatherForecast {
  // ... existing properties
  
  @added(Versions.v2)
  humidity?: int32;
  
  @added(Versions.v2)
  windSpeed?: float32;
}
```

### 6. **Integration with Modern Tools** 🔧

TypeSpec integrates seamlessly with:
- **Microsoft Kiota** - Generate strongly-typed HTTP clients
- **Azure API Management** - Import generated OpenAPI specifications
- **OpenAPI tooling** - Works with existing OpenAPI ecosystem
- **CI/CD pipelines** - Automate spec validation and code generation
- **API gateways** - Deploy contracts to Kong, Envoy, etc.

## The Alternative: Manual API Management 😰

Without TypeSpec, API development typically follows one of two problematic approaches:

### Approach 1: Hand-Written OpenAPI Specifications 📝
**The Process:**
- Write OpenAPI YAML/JSON files manually
- Struggle with verbose, repetitive syntax
- Manually keep specifications in sync with code

**The Problems:**
- **Documentation Drift** - Hand-written specs quickly become outdated
- **Error-Prone** - Typos in massive YAML files break integrations
- **Maintenance Nightmare** - Complex specifications become unwieldy
- **No Validation** - Missing validation rules and inconsistent patterns

### Approach 2: Code-First with Manual Export/Import 🔄
**The Process:**
- Write API implementation first (ASP.NET Core, FastAPI, etc.)
- Export OpenAPI spec from running application
- Manually import spec into API management tools
- Hope frontend teams can work with what's generated

**The Problems:**
- **Implementation Lock-in** - API design driven by code constraints
- **Export/Import Friction** - Manual steps that teams forget or skip
- **Generated Specs** - Often incomplete or poorly documented
- **Coordination Overhead** - Constant communication between teams about API changes
- **Late Discovery** - Integration issues found after implementation

### Common Pain Points Across Both Approaches 😩
**Team Coordination**
- Frontend and backend models diverge over time
- Manual client SDK updates required for every change
- Time wasted on preventable integration issues

**Quality Issues**
- Missing validation rules in specifications
- Inconsistent error handling across endpoints
- Poor documentation that doesn't match reality

**Scaling Challenges**
- Multiple APIs with different standards and patterns
- No reusable components across teams
- Difficult to maintain consistency as organization grows

## TypeSpec in Action: A Quick Example 🎯

Let's see how TypeSpec would improve our weather application from the Aspire series:

**Traditional Approach** (what we had):
```csharp
// Backend C# model
public record WeatherForecast(Guid Id, DateOnly Date, int TemperatureC, string? Summary, string Location);

// Frontend TypeScript interface (manually created)
export interface WeatherForecast {
  id: string;
  date: string;
  temperatureC: number;
  temperatureF: number;
  summary: string;
  location: string;
}
```

**TypeSpec Approach**:
```typespec
model WeatherForecast {
  @format("uuid")
  id: string;
  
  @format("date")
  date: plainDate;
  
  @minimum(-50)
  @maximum(60)
  temperatureC: int32;
  
  temperatureF: int32;
  
  @minLength(1)
  @maxLength(100)
  summary: string;
  
  @minLength(1)
  @maxLength(50)
  location: string;
}

@route("/weather")
namespace WeatherAPI {
  @get
  op getForecast(): {
    @statusCode statusCode: 200;
    @body forecasts: WeatherForecast[];
  };
  
  @post
  op createForecast(@body forecast: WeatherForecast): {
    @statusCode statusCode: 201;
    @body created: WeatherForecast;
  };
}
```

From this single definition:
- Generate OpenAPI spec
- Generate TypeScript types for React
- Generate C# models for .NET
- Generate validation attributes
- Generate API client code
- Generate server stubs

## Installing TypeSpec Tooling 🛠️

Let's get TypeSpec set up in our development environment. We'll need Node.js (which we already have in our Aspire dev container) and the TypeSpec tools.

### Prerequisites 📦

From our previous Aspire tutorial, we already have:
- 🐳 Dev container with Node.js installed
- 🌤️ Our weather application running
- 🔧 VS Code with proper extensions

### Install TypeSpec CLI and Compiler 💻

```bash
# Install TypeSpec globally
npm install -g @typespec/compiler

# Verify installation
tsp --version
```

**What this installs:**
- **TypeSpec Compiler** (`tsp`) - Core compiler for processing `.tsp` files
- **Standard Library** - Built-in types like `string`, `int32`, `utcDateTime`
- **CLI Tools** - Commands for project initialization, compilation, and scaffolding

**Verify Everything Works:**
```bash
# Check available commands
tsp --help

# See installed version and location
tsp --version
which tsp
```

### Install VS Code Extension 🎨

The TypeSpec VS Code extension provides syntax highlighting, IntelliSense, and real-time validation:

1. Open VS Code
2. Go to Extensions (Ctrl+Shift+X)
3. Search for "TypeSpec" 
4. Install the official Microsoft TypeSpec extension

## Adding TypeSpec to Existing Project 📁

Let's add TypeSpec to our existing weather application from the Aspire series. We'll create a new TypeSpec project that will define our API contracts.

### Step 1: Initialize TypeSpec Project 🚀

Navigate to your weather app repository and create a TypeSpec project:

```bash
# From your project root (where you have src/ folder)
mkdir WeatherApp.Typespec
cd WeatherApp.Typespec

# Initialize a new TypeSpec project
tsp init
```

The TypeSpec CLI will guide you through setup:
- **Template**: Choose "Generic Rest API"
- **Package name**: `weather-api-contracts`
- **Description**: "API contracts for weather application"

This creates:
```
WeatherApp.Typespec/
├── package.json
├── tspconfig.yaml
├── main.tsp
└── node_modules/
```

### Step 2: Configure TypeSpec Project 🔧

Update `tspconfig.yaml` to configure our emitters (code generators):

```yaml
emit:
  - "@typespec/openapi3"
options:
  "@typespec/openapi3":
    emitter-output-dir: "{output-dir}/../../generated/openapi"
```

This configuration:
- **`emit`** - Specifies which emitters to run (OpenAPI 3 in this case)
- **`emitter-output-dir`** - Uses relative path to output to `src/generated/openapi` 
- **`{output-dir}`** - TypeSpec's placeholder for the default output directory (tsp-output)
- The `../../` navigates up from `WeatherApp.Typespec/tsp-output` to the project root, then into `src/generated/openapi`

Install the required emitters:

```bash
npm install @typespec/openapi3
```

## Defining Our Weather API Contract 🌤️

Replace the content of `main.tsp` with our complete weather API definition:

```typespec
import "@typespec/http";

using Http;

@service({
  title: "Weather API",
})
@server("http://localhost:5008", "Development server")
namespace WeatherAPI;

@doc("Represents a weather forecast for a specific location and date")
model WeatherForecast {
  @doc("Unique identifier for the weather forecast")
  id: string;
  
  @doc("Date of the weather forecast")
  date: plainDate;
  
  @doc("Temperature in Celsius")
  temperatureC: int32;
  
  @doc("Temperature in Fahrenheit (calculated)")
  temperatureF: int32;
  
  @doc("Weather summary description")
  summary?: string;
  
  @doc("Geographic location of the forecast")
  location: string;
}

@doc("Standard error response")
@error
model ErrorResponse {
  @doc("Error code")
  code: string;

  @doc("Human-readable error message")
  message: string;

  @doc("Additional error details")
  details?: string;

  @doc("Timestamp when error occurred")
  timestamp: utcDateTime;

  @doc("Request ID for tracking")
  requestId?: string;
}

@doc("Bad request error") 
model BadRequestError extends ErrorResponse {
  code: "BAD_REQUEST";
}

@doc("List of weather forecasts")
model WeatherForecastList {
  @doc("Array of weather forecast items")
  items: WeatherForecast[];
}

@route("/weatherforecast")
@tag("Weather")
interface WeatherForecasts {
  /** Get all weather forecasts */
  @get 
  op listForecasts(): {
    @statusCode statusCode: 200;
    @body forecasts: WeatherForecast[];
  } | {
    @statusCode statusCode: 500;
    @body error: ErrorResponse;
  };

  /** Add a new weather forecast */
  @post
  op addForecast(
    @body forecast: WeatherForecast
  ): {
    @statusCode statusCode: 201;
    @body createdForecast: WeatherForecast;
  } | {
    @statusCode statusCode: 400;
    @body error: BadRequestError;
  } | {
    @statusCode statusCode: 500;
    @body error: ErrorResponse;
  };
}
```

**Key Features of This TypeSpec Definition:**

**Server Configuration**
- `@server("http://localhost:5008", "Development server")` - Defines the base URL for the API
- This will be included in the generated OpenAPI specification
- Makes it easy to test the API with tools like Swagger UI

**Clean Error Hierarchy**
- `ErrorResponse` - Base error model with common fields (code, message, details, timestamp, requestId)
- `BadRequestError` - Extends base with specific error code constant
- Easily extensible for additional error types (NotFoundError, ValidationError, etc.)

**Explicit Response Structure**
- Uses `@statusCode` and `@body` decorators for clear response definitions
- Union types (`|`) to define success and error responses
- Makes the API contract explicit and unambiguous

**Interface-Based Operations**
- `interface WeatherForecasts` groups related operations
- Cleaner than namespace-based operations
- Better IntelliSense and tooling support in VS Code

## Creating OpenAPI File with Emitter 📄

Now let's generate the OpenAPI specification from our TypeSpec definition:

```bash
# From the WeatherApp.Typespec directory
tsp compile .
```

This generates:
```
src/
└── generated/
    └── openapi/
        └── openapi.yaml
```

The generated `openapi.yaml` will contain a complete OpenAPI 3.0 specification with:
- All endpoints with proper HTTP methods
- Request/response schemas
- Validation rules
- Error responses
- Documentation from our `@doc` decorators

### Verify Generated OpenAPI

Let's check what we generated:

```bash
# View the generated OpenAPI spec
cat src/generated/openapi/openapi.yaml
```

You'll see a fully-formed OpenAPI specification that's much cleaner and more comprehensive than what we could write by hand!

## Adding Swagger to AppHost 📊

Let's integrate Swagger UI into our Aspire application so we can view and test our API specification.

Update `WeatherApp.AppHost/AppHost.cs` to include a Swagger UI container:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

#pragma warning disable ASPIRECOSMOSDB001

var cosmos = builder.AddAzureCosmosDB("cosmos").RunAsPreviewEmulator(
    emulator =>
    {
        emulator.WithDataExplorer();            
    });

#pragma warning restore ASPIRECOSMOSDB001

var database = cosmos.AddCosmosDatabase("WeatherDB");
var container = database.AddContainer("WeatherData", "/location");

var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi")
    .WithReference(container);

var seeder = builder.AddProject<Projects.WeatherApp_Seed>("weatherseeder")
    .WithReference(container)
    .WithExplicitStart();

var swagger = builder.AddContainer("swagger-ui", "swaggerapi/swagger-ui")
    .WithBindMount("../generated/openapi/openapi.yaml", "/usr/share/nginx/html/openapi.yaml")
    .WithEnvironment("SWAGGER_JSON_URL", "/openapi.yaml")
    .WithHttpEndpoint(targetPort: 8080, name: "swagger");

var frontend = builder.AddViteApp("frontend", "../WeatherApp.Web")
    .WithNpmPackageInstallation()
    .WithReference(api)
    .WithEnvironment("BROWSER", "false")
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

**Key Configuration Details:**

**Swagger UI Container**
- Uses the official `swaggerapi/swagger-ui` Docker image
- **`WithBindMount`** - Mounts the generated OpenAPI file directly to the Swagger UI container
- **`WithEnvironment("SWAGGER_JSON_URL", "/openapi.yaml")`** - Tells Swagger UI where to find the spec
- **`WithHttpEndpoint(targetPort: 8080)`** - Exposes Swagger UI on port 8080

**Cosmos DB Partition Key**
- Updated to use `/location` as the partition key (matching our WeatherForecast model)
- Previously was `/id` which wasn't optimal for querying by location

**Service Ordering**
- API and seeder are defined before Swagger
- Ensures the OpenAPI file is generated before Swagger tries to load it
- Swagger appears in the Aspire dashboard with the other services

## Creating Controllers with Emitter 🏗️

Now let's create a TypeSpec emitter that generates .NET controllers from our specification.

### Step 1: Install ASP.NET Core Emitter 📦

```bash
# From WeatherApp.Typespec directory
npm install @typespec/http-server-csharp
```

### Step 2: Update TypeSpec Configuration ⚙️

Update `tspconfig.yaml` to include the C# emitter:

```yaml
emit:
  - "@typespec/openapi3"
  - "@typespec/http-server-csharp"
options:
  "@typespec/openapi3":
    emitter-output-dir: "{output-dir}/../../generated/openapi"
  "@typespec/http-server-csharp":
    emitter-output-dir: "{output-dir}/../../WeatherApp.Api"
    emit-mocks: none
```

**Key Configuration Options:**

- **`emitter-output-dir: "{output-dir}/../../WeatherApp.Api"`** - Generates C# code directly into the API project
- **`emit-mocks: none`** - Disables mock generation, we only want the models and interfaces
- This approach integrates generated code directly into your existing project structure

### Step 3: Generate Controllers 🤖

```bash
# Regenerate with new emitter
tsp compile .
```

This creates files directly in your API project:
```
src/
├── WeatherApp.Api/
│   ├── generated/           # TypeSpec-generated code
│   │   ├── Models/
│   │   │   ├── WeatherForecast.cs
│   │   │   ├── ErrorResponse.cs
│   │   │   └── BadRequestError.cs
│   │   └── ... (other generated files)
│   └── ... (other API files)
└── generated/
    └── openapi/
        └── openapi.yaml
```

### Step 4: Clean Up Data Project 🧹

Now that we're using TypeSpec-generated code, we can remove some files from the Data project that are no longer needed:

**Remove these files from `WeatherApp.Data/`:**
- `ServiceExtensions.cs` - Service registration is now handled by generated code
- `WeatherService.cs` / `IWeatherService` - Business logic will use generated models directly

The generated C# server code from TypeSpec includes:
- Models that match your TypeSpec definitions
- Interfaces for operations
- Proper serialization attributes

**Keep in `WeatherApp.Data/`:**
- `WeatherContext.cs` - Entity Framework DbContext for database access
- `WeatherForecast.cs` - Can be replaced with the generated model or kept for EF-specific configurations

### Step 5: Update API to Use Generated Models 📦

Update your `WeatherApp.Api/Program.cs` to use the generated models:

```csharp
using WeatherApi;
using WeatherApp.Data;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring OpenAPI at https://aka.ms/aspnet/openapi
builder.Services.AddOpenApi();
builder.AddServiceDefaults();

// Add Entity Framework with Cosmos DB - connection is injected automatically by Aspire
builder.AddCosmosDbContext<WeatherContext>("WeatherData");

// Register weather services from the Data project
builder.Services.AddScoped<IWeatherForecasts, WeatherService>();
builder.Services.AddControllers();

var app = builder.Build();

app.UseHttpsRedirection();
app.MapControllers();
app.Run();
```

**Key Points:**
- **`IWeatherForecasts`** - Interface generated from TypeSpec operations
- **`WeatherService`** - Your implementation of the generated interface
- **`AddOpenApi()`** - Uses built-in .NET OpenAPI support instead of Swashbuckle
- **Simplified** - No manual Swagger configuration, leveraging Aspire and generated code

### Step 6: Implement the Generated Interface 🔌

Create `WeatherApp.Api/WeatherService.cs` to implement the TypeSpec-generated interface:

```csharp
using System.Text.Json.Nodes;
using WeatherApi;
using WeatherApp.Data;
using Microsoft.EntityFrameworkCore;

public class WeatherService : IWeatherForecasts
{
    private readonly WeatherContext _context;

    public WeatherService(WeatherContext context)
    {
        _context = context;
    }

    public async Task<JsonNode> AddForecastAsync(WeatherApi.WeatherForecast body)
    {
        var entity = new WeatherApp.Data.WeatherForecast(
            Guid.NewGuid(),
            DateOnly.FromDateTime(body.Date),
            body.TemperatureC,
            body.Summary,
            body.Location
        );
        _context.WeatherForecasts.Add(entity);
        await _context.SaveChangesAsync();
        return System.Text.Json.JsonSerializer.SerializeToNode(entity) ?? new JsonObject();
    }

    public async Task<JsonNode> ListForecastsAsync()
    {
        var forecasts = await _context.WeatherForecasts.ToListAsync();
        var json = System.Text.Json.JsonSerializer.Serialize(forecasts);
        var node = JsonNode.Parse(json);
        return node ?? new JsonArray();
    }
}
```

**Implementation Details:**
- **`IWeatherForecasts`** - Generated interface from TypeSpec with `ListForecastsAsync()` and `AddForecastAsync()` methods
- **`WeatherApi.WeatherForecast`** - Generated model from TypeSpec used for API contract
- **`WeatherApp.Data.WeatherForecast`** - Entity Framework model for database persistence
- **Type Mapping** - Converts between API models and database entities
- **`JsonNode` Returns** - Generated interface uses `JsonNode` for flexible response handling

**Why Two Models?**
- **API Model** (`WeatherApi.WeatherForecast`) - Generated from TypeSpec, matches API contract exactly
- **Database Model** (`WeatherApp.Data.WeatherForecast`) - Entity Framework entity with database-specific attributes
- This separation allows the API contract to evolve independently from database schema

### Step 7: Update Seed Project 🌱

The seed project can use Entity Framework directly without needing the TypeSpec-generated service layer. Update `WeatherApp.Seed/Program.cs`:

```csharp
using WeatherApp.Data;
using WeatherApp.Seed;

var builder = Host.CreateApplicationBuilder(args);

builder.AddServiceDefaults();

// Add Entity Framework with Cosmos DB
builder.AddCosmosDbContext<WeatherContext>("WeatherData");

// Register the worker service that seeds data
builder.Services.AddHostedService<Worker>();

var host = builder.Build();
host.Run();
```

**Simplified Seed Project:**
- **Direct EF Access** - Worker service injects `WeatherContext` directly
- **No API Layer** - Seed project doesn't need the API contract or generated services
- **Database-Only** - Works with `WeatherApp.Data.WeatherForecast` entities directly
- **Separation of Concerns** - TypeSpec contract is for API consumers, not internal tooling

Update `WeatherApp.Seed/Worker.cs` to use Entity Framework directly:

```csharp
using Microsoft.EntityFrameworkCore;
using WeatherApp.Data;

namespace WeatherApp.Seed;

public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly IServiceScopeFactory _serviceScopeFactory;
    private readonly IHostApplicationLifetime _hostApplicationLifetime;

    public Worker(ILogger<Worker> logger, IServiceScopeFactory serviceScopeFactory, 
                  IHostApplicationLifetime hostApplicationLifetime)
    {
        _logger = logger;
        _serviceScopeFactory = serviceScopeFactory;
        _hostApplicationLifetime = hostApplicationLifetime;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Weather Seeder starting at: {time}", DateTimeOffset.Now);
        
        using var scope = _serviceScopeFactory.CreateScope();
        var weatherContext = scope.ServiceProvider.GetRequiredService<WeatherContext>();
        
        // Check if data already exists
        var firstForecast = await weatherContext.WeatherForecasts
            .FirstOrDefaultAsync(stoppingToken);
        if (firstForecast != null)
        {
            _logger.LogInformation("Weather data already exists. Skipping seed.");
            _hostApplicationLifetime.StopApplication();
            return;
        }

        // Seed fake weather data
        _logger.LogInformation("Seeding weather data...");
        
        var summaries = new[]
        {
            "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", 
            "Balmy", "Hot", "Sweltering", "Scorching"
        };

        var cities = new[]
        {
            "New York, USA", "London, UK", "Tokyo, Japan", "Sydney, Australia", 
            "Paris, France", "Berlin, Germany", "Toronto, Canada", "Mumbai, India",
            "São Paulo, Brazil", "Cairo, Egypt", "Moscow, Russia", "Beijing, China",
            "Mexico City, Mexico", "Lagos, Nigeria", "Bangkok, Thailand", "Dubai, UAE",
            "Singapore", "Buenos Aires, Argentina", "Stockholm, Sweden", 
            "Amsterdam, Netherlands"
        };

        var forecasts = Enumerable.Range(1, 5).Select(index =>
            new WeatherForecast
            (
                Guid.NewGuid(),
                DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
                Random.Shared.Next(-20, 55),
                summaries[Random.Shared.Next(summaries.Length)],
                cities[Random.Shared.Next(cities.Length)]
            ))
            .ToArray();

        // Add all forecasts to the context
        await weatherContext.WeatherForecasts.AddRangeAsync(forecasts, stoppingToken);
        await weatherContext.SaveChangesAsync(stoppingToken);

        foreach (var forecast in forecasts)
        {
            _logger.LogInformation("Added forecast: {Date} - {Location} - {Summary} - {TempC}°C", 
                forecast.Date, forecast.Location, forecast.Summary, forecast.TemperatureC);
        }
        
        _logger.LogInformation("Weather Seeder completed seeding {Count} forecasts at: {time}", 
            forecasts.Length, DateTimeOffset.Now);
        
        // Stop the application after seeding is complete
        _hostApplicationLifetime.StopApplication();
    }
}
```

**Worker Service Implementation:**
- **Direct DbContext Injection** - Uses `IServiceScopeFactory` to create a scope and get `WeatherContext`
- **Idempotent Seeding** - Checks if data exists before seeding to avoid duplicates
- **Database Entities** - Works directly with `WeatherApp.Data.WeatherForecast` record
- **Auto-Shutdown** - Stops the application after seeding completes (or if data exists)
- **Cosmos DB Compatible** - Uses `FirstOrDefaultAsync()` instead of `Any()` for better Cosmos performance

This demonstrates a key architectural benefit: internal tools (like seeders) can work directly with the database layer, while external consumers use the TypeSpec-defined API contract.

## Testing the Complete Setup 🧪

### Step 1: Generate and Run 🚀

```bash
# Generate all contracts and code
cd WeatherApp.Typespec
tsp compile .

# Run the Aspire application
cd ..
aspire run
```

### Step 2: View Results 👁️

Your Aspire dashboard now shows:
- **Weather API** - Your .NET API with generated controllers
- **Frontend** - React app consuming the API
- **Swagger UI** - Interactive API documentation at http://localhost:8080
- **Cosmos DB** - Database with seeded data

### Step 3: Test the API ✅

Visit the Swagger UI to:
- View complete API documentation
- Test endpoints interactively
- See validation rules in action
- Verify error responses

## GitHub Repository & Resources 📚

### Complete Example Repository

All code from this tutorial is available at:
**[github.com/two4suited/blog-platform-typespec](https://github.com/two4suited/blog-platform-typespec/tree/aspire-tools-typespec)**

Branch: `aspire-tools-typespec`

The repository includes:
- Complete TypeSpec definitions
- Generated OpenAPI specifications  
- .NET API with generated models
- React frontend with TypeScript types
- Aspire orchestration with Swagger UI
- CI/CD pipeline examples

### Folder Structure
```
blog-platform-typespec/
├── WeatherApp.Typespec/     # TypeSpec definitions
│   ├── main.tsp
│   ├── tspconfig.yaml
│   └── package.json
├── src/                     # Aspire application
│   ├── generated/           # Generated OpenAPI specs
│   │   └── openapi/
│   ├── WeatherApp.Api/      # API project
│   │   └── generated/       # TypeSpec-generated models
│   │       └── Models/
│   ├── WeatherApp.Web/
│   └── WeatherApp.AppHost/
└── README.md
```

## Additional Resources 🔗

### Official Documentation
- [TypeSpec Documentation](https://typespec.io/) - Complete language reference and guides
- [TypeSpec GitHub](https://github.com/microsoft/typespec) - Source code and issues
- [TypeSpec Playground](https://typespec.io/playground) - Try TypeSpec in your browser
- [TypeSpec VS Code Extension](https://marketplace.visualstudio.com/items?itemName=Microsoft.typespec-vscode) - Official editor support

### Emitters and Tools
- [OpenAPI 3 Emitter](https://typespec.io/docs/emitters/openapi3/reference) - Generate OpenAPI specs
- [HTTP Server C# Emitter](https://typespec.io/docs/emitters/http-server-csharp) - Generate .NET controllers
- [TypeScript Client Emitter](https://typespec.io/docs/emitters/typescript) - Generate TypeScript clients
- [JSON Schema Emitter](https://typespec.io/docs/emitters/json-schema) - Generate JSON schemas

### Community Resources
- [TypeSpec Samples](https://github.com/microsoft/typespec/tree/main/packages/samples) - Example TypeSpec definitions
- [TypeSpec Discord](https://discord.gg/typespec) - Community discussion and support
- [Azure REST API Specs](https://github.com/Azure/azure-rest-api-specs) - Real-world TypeSpec examples from Azure
- [Microsoft Graph TypeSpec](https://github.com/microsoftgraph/msgraph-typespec) - Large-scale TypeSpec implementation

### Learning Path
1. **Start Here**: [TypeSpec Getting Started](https://typespec.io/docs/getting-started)
2. **Language Tour**: [TypeSpec Language Basics](https://typespec.io/docs/language-basics)
3. **HTTP APIs**: [REST API Guide](https://typespec.io/docs/libraries/http)
4. **Advanced**: [Custom Decorators](https://typespec.io/docs/extending-typespec/basics)

## Conclusion 🎉

We've successfully transformed our weather application from implementation-first to contract-first development using TypeSpec. By defining our API contract in TypeSpec, we've eliminated the manual coordination between frontend and backend teams while gaining better type safety, comprehensive documentation, and automated code generation.

**What We Built:**
- ✅ **TypeSpec API Definition** - Single source of truth with clean, readable syntax
- ✅ **OpenAPI Specification** - Auto-generated from TypeSpec for tooling compatibility
- ✅ **C# Server Code** - Generated interfaces and models for .NET API
- ✅ **Interactive Documentation** - Swagger UI integrated with Aspire dashboard
- ✅ **End-to-End Type Safety** - Contract drives both frontend and backend implementation

**Key Benefits:**
- **No More Drift** - Frontend and backend can't get out of sync
- **Parallel Development** - Teams work from the same contract simultaneously
- **Better Quality** - Validation and error handling defined upfront
- **Future-Proof** - Works with existing OpenAPI ecosystem and tools

Our weather API now demonstrates contract-first development that can scale across teams and projects. The TypeSpec definition becomes the authoritative source that drives both implementation and documentation.

### Next Steps in Our Platform Journey

Now that we have a robust local development environment with Aspire and contract-first APIs with TypeSpec, it's time to think about deploying our application to production. In our next post, we'll explore how to use **Terraform** to create the cloud infrastructure needed to host our weather application at scale.

---

**Coming Up Next**: [Infrastructure as Code with Terraform: Deploying Our Platform to Azure](https://brianpsheridan.com/categories.html#platform) ☁️