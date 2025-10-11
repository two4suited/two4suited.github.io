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
- **Azure API Management** - Import TypeSpec specs directly
- **OpenAPI tooling** - Works with existing OpenAPI ecosystem
- **CI/CD pipelines** - Automate spec validation and code generation
- **API gateways** - Deploy contracts to Kong, Envoy, etc.

## The Alternative: Manual API Management 😰

Without TypeSpec, API development looks like:

**Documentation Drift**
- OpenAPI specs written by hand, quickly become outdated
- Frontend and backend models diverge
- Integration bugs discovered late in development

**Coordination Overhead**
- Constant communication between teams about API changes
- Manual client SDK updates
- Time wasted on preventable integration issues

**Error-Prone Process**
- Typos in hand-written OpenAPI specs
- Missing validation rules
- Inconsistent error handling across endpoints

**Scaling Challenges**
- Multiple APIs with different standards
- No reusable patterns across teams
- Difficult to maintain consistency

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

### Prerequisites
From our previous Aspire tutorial, we already have:
- Dev container with Node.js installed
- Our weather application running
- VS Code with proper extensions

### Install TypeSpec CLI and Compiler

```bash
# Install TypeSpec globally
npm install -g @typespec/compiler

# Verify installation
tsp --version
```

### Install VS Code Extension

The TypeSpec VS Code extension provides syntax highlighting, IntelliSense, and real-time validation:

1. Open VS Code
2. Go to Extensions (Ctrl+Shift+X)
3. Search for "TypeSpec" 
4. Install the official Microsoft TypeSpec extension

## Adding TypeSpec to Existing Project 📁

Let's add TypeSpec to our existing weather application from the Aspire series. We'll create a new TypeSpec project that will define our API contracts.

### Step 1: Initialize TypeSpec Project

Navigate to your weather app repository and create a TypeSpec project:

```bash
# From your project root (where you have src/ folder)
mkdir api-contracts
cd api-contracts

# Initialize a new TypeSpec project
tsp init
```

The TypeSpec CLI will guide you through setup:
- **Template**: Choose "Generic Rest API"
- **Package name**: `weather-api-contracts`
- **Description**: "API contracts for weather application"

This creates:
```
api-contracts/
├── package.json
├── tspconfig.yaml
├── main.tsp
└── node_modules/
```

### Step 2: Configure TypeSpec Project

Update `tspconfig.yaml` to configure our emitters (code generators):

```yaml
extends: "@typespec/http/all"
parameters:
  "service-name": "WeatherAPI"
emitters:
  "@typespec/openapi3":
    output-dir: "../generated/openapi"
  "@typespec/json-schema": 
    output-dir: "../generated/schemas"
options:
  "@typespec/openapi3":
    file-type: "yaml"
```

Install the required emitters:

```bash
npm install @typespec/openapi3 @typespec/json-schema
```

## Defining Our Weather API Contract 🌤️

Replace the content of `main.tsp` with our weather API definition:

```typespec
import "@typespec/http";
import "@typespec/rest";
import "@typespec/openapi3";

using TypeSpec.Http;
using TypeSpec.Rest;

@service({
  title: "Weather API",
  description: "A service for managing weather forecasts",
  version: "1.0.0",
})
@server("https://localhost:7097", "Development server")
namespace WeatherAPI;

@doc("Represents a weather forecast for a specific location and date")
model WeatherForecast {
  @doc("Unique identifier for the forecast")
  @format("uuid")
  id: string;

  @doc("Date of the forecast")  
  @format("date")
  date: plainDate;

  @doc("Temperature in Celsius")
  @minimum(-50)
  @maximum(60)
  temperatureC: int32;

  @doc("Temperature in Fahrenheit (calculated)")
  temperatureF: int32;

  @doc("Weather summary description")
  @minLength(1)
  @maxLength(100)
  summary: string;

  @doc("Location for this forecast")
  @minLength(1)
  @maxLength(50)
  location: string;
}

@doc("Request model for creating a new weather forecast")
model CreateWeatherRequest {
  @doc("Date of the forecast")
  @format("date")  
  date: plainDate;

  @doc("Temperature in Celsius")
  @minimum(-50)
  @maximum(60)
  temperatureC: int32;

  @doc("Weather summary description")
  @minLength(1)
  @maxLength(100)
  summary: string;

  @doc("Location for this forecast")
  @minLength(1)
  @maxLength(50)
  location: string;
}
```

## Adding Custom Error Messages 🚨

Let's add comprehensive error handling to our TypeSpec definition:

```typespec
// Add this to main.tsp after the models

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

@doc("Validation error details")
model ValidationError extends ErrorResponse {
  @doc("Field-specific validation errors")
  fieldErrors?: Record<string[]>;
}

@doc("Not found error")
model NotFoundError extends ErrorResponse {
  code: "NOT_FOUND";
}

@doc("Bad request error") 
model BadRequestError extends ErrorResponse {
  code: "BAD_REQUEST";
}

// Weather API operations with comprehensive error handling
@tag("Weather")
@route("/weather")
namespace Weather {
  @doc("Get all weather forecasts")
  @get
  op listForecasts(): {
    @statusCode statusCode: 200;
    @body forecasts: WeatherForecast[];
  } | {
    @statusCode statusCode: 500;
    @body error: ErrorResponse;
  };

  @doc("Get a specific weather forecast by ID")
  @get
  op getForecast(@path id: string): {
    @statusCode statusCode: 200;
    @body forecast: WeatherForecast;
  } | {
    @statusCode statusCode: 404;
    @body error: NotFoundError;
  } | {
    @statusCode statusCode: 400;
    @body error: BadRequestError;
  };

  @doc("Create a new weather forecast")
  @post
  op createForecast(@body request: CreateWeatherRequest): {
    @statusCode statusCode: 201;
    @body forecast: WeatherForecast;
  } | {
    @statusCode statusCode: 400;
    @body error: ValidationError;
  } | {
    @statusCode statusCode: 500;
    @body error: ErrorResponse;
  };

  @doc("Update an existing weather forecast")
  @put
  op updateForecast(@path id: string, @body request: CreateWeatherRequest): {
    @statusCode statusCode: 200;
    @body forecast: WeatherForecast;
  } | {
    @statusCode statusCode: 404;
    @body error: NotFoundError;
  } | {
    @statusCode statusCode: 400;
    @body error: ValidationError;
  };

  @doc("Delete a weather forecast")
  @delete
  op deleteForecast(@path id: string): {
    @statusCode statusCode: 204;
  } | {
    @statusCode statusCode: 404;
    @body error: NotFoundError;
  };
}
```

## Creating OpenAPI File with Emitter 📄

Now let's generate the OpenAPI specification from our TypeSpec definition:

```bash
# From the api-contracts directory
tsp compile .
```

This generates:
```
generated/
├── openapi/
│   └── openapi.yaml
└── schemas/
    └── *.json
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
cat ../generated/openapi/openapi.yaml
```

You'll see a fully-formed OpenAPI specification that's much cleaner and more comprehensive than what we could write by hand!

## Adding Swagger to AppHost 📊

Let's integrate Swagger UI into our Aspire application so we can view and test our API specification.

### Step 1: Add Swagger Support to AppHost

Update your `WeatherApp.AppHost/WeatherApp.AppHost.csproj` to include the OpenAPI file as content:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- existing content -->
  
  <ItemGroup>
    <Content Include="../generated/openapi/openapi.yaml">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
</Project>
```

### Step 2: Add Swagger UI Container to AppHost

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
var container = database.AddContainer("WeatherData", "/id");

// Add Swagger UI for API documentation
var swagger = builder.AddContainer("swagger-ui", "swaggerapi/swagger-ui")
    .WithEnvironment("SWAGGER_JSON_URL", "http://localhost:8080/openapi.yaml")
    .WithBindMount("../generated/openapi", "/usr/share/nginx/html")
    .WithHttpEndpoint(port: 8080, targetPort: 8080, name: "swagger");

// Pass Cosmos DB container connection to the API
var api = builder.AddProject<Projects.WeatherApp_Api>("weatherapi")
    .WithReference(container);

// Add explicit dependency so Swagger can find the OpenAPI file
swagger.WithReference(api);

// Add the React frontend using Vite integration
var frontend = builder.AddViteApp("frontend", "../WeatherApp.Web")
    .WithNpmPackageInstallation()
    .WithReference(api)
    .WithEnvironment("BROWSER", "false")
    .WithExternalHttpEndpoints();

// Add the data seeding worker service with explicit start
var seeder = builder.AddProject<Projects.WeatherApp_Seed>("weatherseeder")
    .WithReference(container)
    .WithExplicitStart();

builder.Build().Run();
```

## Creating Controllers with Emitter 🏗️

Now let's create a TypeSpec emitter that generates .NET controllers from our specification.

### Step 1: Install ASP.NET Core Emitter

```bash
# From api-contracts directory
npm install @typespec/http-server-csharp
```

### Step 2: Update TypeSpec Configuration

Update `tspconfig.yaml` to include the C# emitter:

```yaml
extends: "@typespec/http/all"
parameters:
  "service-name": "WeatherAPI"
emitters:
  "@typespec/openapi3":
    output-dir: "../generated/openapi"
  "@typespec/json-schema": 
    output-dir: "../generated/schemas"
  "@typespec/http-server-csharp":
    output-dir: "../generated/controllers"
    namespace: "WeatherApp.Generated"
options:
  "@typespec/openapi3":
    file-type: "yaml"
```

### Step 3: Generate Controllers

```bash
# Regenerate with new emitter
tsp compile .
```

This creates:
```
generated/
├── openapi/
├── schemas/
└── controllers/
    ├── Models/
    │   ├── WeatherForecast.cs
    │   ├── CreateWeatherRequest.cs
    │   └── ErrorResponse.cs
    └── Controllers/
        └── WeatherController.cs
```

### Step 4: Integration with Existing API

Add the generated models to your existing API project:

```bash
# From your API project
dotnet add package Microsoft.AspNetCore.Mvc.Abstractions

# Copy generated files or add them as linked files
```

Update your `WeatherApp.Api/Program.cs` to reference generated models:

```csharp
using WeatherApp.Data;
using WeatherApp.Generated.Models; // Generated models

var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.AddCosmosDbContext<WeatherContext>("WeatherData");
builder.Services.AddWeatherServices();

// Add Swagger with our generated OpenAPI spec
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo 
    { 
        Title = "Weather API", 
        Version = "v1",
        Description = "Generated from TypeSpec"
    });
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();

var app = builder.Build();

app.MapDefaultEndpoints();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "Weather API V1");
        c.RoutePrefix = "swagger";
    });
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## Testing the Complete Setup 🧪

### Step 1: Generate and Run

```bash
# Generate all contracts and code
cd api-contracts
tsp compile .

# Run the Aspire application
cd ..
aspire run
```

### Step 2: View Results

Your Aspire dashboard now shows:
- **Weather API** - Your .NET API with generated controllers
- **Frontend** - React app consuming the API
- **Swagger UI** - Interactive API documentation at http://localhost:8080
- **Cosmos DB** - Database with seeded data

### Step 3: Test the API

Visit the Swagger UI to:
- View complete API documentation
- Test endpoints interactively
- See validation rules in action
- Verify error responses

## Automation and CI/CD Integration 🔄

### Step 1: Add Build Script

Create `api-contracts/package.json` script for automation:

```json
{
  "scripts": {
    "build": "tsp compile .",
    "watch": "tsp compile . --watch",
    "validate": "tsp compile . --no-emit"
  }
}
```

### Step 2: Pre-build Hook

Add to your main project's build process:

```bash
# Before building the API, regenerate contracts
cd api-contracts && npm run build && cd ..
dotnet build
```

### Step 3: CI/CD Integration

```yaml
# GitHub Actions example
- name: Generate API Contracts
  run: |
    cd api-contracts
    npm install
    npm run build
    
- name: Build API with Generated Code
  run: dotnet build src/WeatherApp.Api
```

## GitHub Repository & Resources 📚

### Complete Example Repository

All code from this tutorial is available at:
**[github.com/two4suited/blog-platform-typespec](https://github.com/two4suited/blog-platform-typespec)**

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
├── api-contracts/           # TypeSpec definitions
│   ├── main.tsp
│   ├── tspconfig.yaml
│   └── package.json
├── generated/               # Generated artifacts
│   ├── openapi/
│   ├── schemas/
│   └── controllers/
├── src/                     # Aspire application
│   ├── WeatherApp.Api/
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

## What We've Accomplished Today 🎉

**🏗️ Complete TypeSpec Integration**
- ✅ Installed TypeSpec tooling and VS Code extension
- ✅ Created TypeSpec project with proper configuration
- ✅ Defined comprehensive API contracts with validation
- ✅ Added custom error messages and response handling

**📄 Code Generation Pipeline**
- ✅ Generated OpenAPI 3.0 specification automatically
- ✅ Created .NET controllers and models from TypeSpec
- ✅ Integrated Swagger UI into Aspire dashboard
- ✅ Set up automated contract-first development workflow

**🔗 Full Integration**
- ✅ Connected TypeSpec to existing Aspire application
- ✅ Added interactive API documentation
- ✅ Established single source of truth for API contracts
- ✅ Created sustainable development process

## Conclusion 🎉

We've successfully transformed our weather application from implementation-first to contract-first development using TypeSpec. By defining our API contract in TypeSpec, we've eliminated the manual coordination between frontend and backend teams while gaining better type safety, comprehensive documentation, and automated code generation.

**What We Accomplished:**
- ✅ **Contract-First Development** - Single source of truth for our weather API
- ✅ **Automated Code Generation** - OpenAPI specs, .NET controllers, and TypeScript types
- ✅ **Comprehensive Error Handling** - Proper validation and error responses
- ✅ **Interactive Documentation** - Swagger UI integrated with Aspire dashboard
- ✅ **Type Safety** - End-to-end type safety from contract to implementation
- ✅ **Developer Experience** - IntelliSense, validation, and automated workflows

**Why This Approach Works:**
- **Eliminates Drift** - Frontend and backend can't get out of sync
- **Faster Development** - Teams can work in parallel from the same contract
- **Better Quality** - Validation and error handling defined upfront
- **Future-Proof** - Works with existing OpenAPI ecosystem and tools

Our weather API now serves as a model for contract-first development that can scale across teams and projects. The TypeSpec definition becomes the authoritative source that drives both implementation and documentation.

### Next Steps in Our Platform Journey

Now that we have a robust local development environment with Aspire and contract-first APIs with TypeSpec, it's time to think about deploying our application to production. In our next post, we'll explore how to use **Terraform** to create the cloud infrastructure needed to host our weather application at scale.

---

**Coming Up Next**: [Infrastructure as Code with Terraform: Deploying Our Platform to Azure](https://brianpsheridan.com/categories.html#platform) ☁️