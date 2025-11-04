---
title: "Building a Modern Development Platform: Kiota for Multi-Language API Clients 🔧"
date: 2025-11-04T06:00:00-07:00
draft: false
categories: ["platform","kiota","api","client-generation","sdk"]
description: "Exploring Kiota for generating type-safe API clients in multiple languages with flexible authentication patterns and minimal dependencies"
---

[Series Posts](https://brianpsheridan.com/categories.html#platform)

> 💻 **Source Code:** The complete code for this post is available in the [`aspire-tools-kiota`](https://github.com/two4suited/blog-platform-aspire/tree/aspire-tools-kiota) branch of our GitHub repository.

## Introduction 🚀

In our [TypeSpec post](https://brianpsheridan.com/platform/typespec/api/contract-first/openapi/2025/10/14/tools-typespec.html), we defined our Weather API contract and generated an OpenAPI specification. Now we have a machine-readable API description, but how do frontend and backend teams consume it? Should they manually write HTTP client code? Parse JSON responses into TypeScript interfaces by hand? Hope the API contract doesn't change?

This is where **Kiota** comes in. Kiota is Microsoft's next-generation API client generator that produces clean, idiomatic, type-safe client code from your OpenAPI specifications. Instead of manually writing HTTP requests and response parsing, you get a fully typed SDK that feels native to your language of choice.

## What is Kiota? 🤔

[Kiota](https://learn.microsoft.com/en-us/openapi/kiota/overview) is an OpenAPI-based API client generator that creates lightweight, strongly-typed HTTP clients for multiple programming languages. Unlike traditional SDK generators that produce bloated code with heavy dependencies, Kiota generates minimal, focused clients that only include what you need.

**Key Characteristics:**

- 🎯 **Type-Safe**: Generated clients are strongly typed based on your OpenAPI spec
- 🪶 **Minimal Dependencies**: Only includes required libraries, not entire HTTP frameworks
- 🌍 **Multi-Language**: Supports C#, TypeScript, Java, Python, Go, PHP, and more
- 🔐 **Authentication Flexible**: Easy to plug in OAuth, API keys, or custom auth providers
- 📦 **OpenAPI Native**: Works with any OpenAPI 3.0+ specification
- ⚡ **Modern Patterns**: Native async/await, proper error handling, and cancellation support

### How Kiota Differs from Other Generators 🔄

**Traditional SDK Generators (like Swagger Codegen):**
```typescript
// Heavy dependencies, bloated code
import { DefaultApi, Configuration } from './generated-sdk';

const config = new Configuration({
  basePath: 'https://api.example.com',
  // Lots of configuration boilerplate
});

const api = new DefaultApi(config);
const response = await api.getWeatherForecast();
```

**Kiota-Generated Client:**
```typescript
// Clean, minimal, idiomatic code
import { AnonymousAuthenticationProvider } from '@microsoft/kiota-abstractions';
import { FetchRequestAdapter } from '@microsoft/kiota-http-fetchlibrary';
import { createWeatherApiClient } from './client';

const authProvider = new AnonymousAuthenticationProvider();
const adapter = new FetchRequestAdapter(authProvider);
adapter.baseUrl = '/api';
const client = createWeatherApiClient(adapter);
const forecasts = await client.weather.get();
```

The difference? **Kiota generates code that feels hand-written**, not machine-generated.

## Why Use Kiota? 💡

### 1. **Contract-First Development** 📋

Kiota completes the contract-first workflow we started with TypeSpec:

1. 📝 Define API in TypeSpec
2. 🔄 Generate OpenAPI spec
3. 🤖 Generate clients with Kiota
4. 🚀 Distribute SDKs to consuming teams

Both frontend and backend teams work from the same source of truth. Changes to the API contract are immediately reflected in generated clients.

### 2. **Type Safety Across Languages** 🔒

Your OpenAPI models become native types:

**OpenAPI Spec:**
```yaml
components:
  schemas:
    WeatherForecast:
      type: object
      required:
        - id
        - date
        - temperatureC
      properties:
        id:
          type: string
        date:
          type: string
          format: date
        temperatureC:
          type: integer
```

**Generated TypeScript:**
```typescript
export interface WeatherForecast {
    id: string;
    date: Date;
    temperatureC: number;
    temperatureF?: number;
    summary?: string;
}
```

**Generated C#:**
```csharp
public class WeatherForecast 
{
    public required string Id { get; set; }
    public required DateOnly Date { get; set; }
    public required int TemperatureC { get; set; }
    public int? TemperatureF { get; set; }
    public string? Summary { get; set; }
}
```

No manual typing. No drift. Compiler-enforced correctness.

### 3. **Flexible Authentication** 🔐

Kiota makes authentication pluggable. Whether you're using:
- 🔑 OAuth 2.0 / OpenID Connect
- 🎫 API Keys
- 🔐 Bearer tokens
- 🛡️ Custom authentication schemes

You can easily configure the authentication provider:

```typescript
// OAuth example
const authProvider = new MyOAuthProvider(/* config */);
const adapter = new FetchRequestAdapter(authProvider);
const client = createWeatherApiClient(adapter);
```

### 4. **Minimal Footprint** 🪶

Traditional SDK generators create monolithic clients with everything included. Kiota uses a layered approach:

- **Abstractions Layer**: Core interfaces (minimal)
- **HTTP Library**: Pluggable (use fetch, axios, etc.)
- **Serialization**: JSON support (lightweight)
- **Generated Code**: Only what your API needs

**Result:** Much smaller bundle sizes for web applications.

### 5. **Language-Native Patterns** 🌍

Kiota generates code that follows each language's best practices:

**TypeScript/JavaScript:**
- Promise-based async
- Proper error handling
- Tree-shakeable imports

**C#:**
- async/await patterns
- IAsyncEnumerable for collections
- Nullable reference types
- CancellationToken support

**Python:**
- Type hints
- Async/await with asyncio
- Context managers

**Java:**
- CompletableFuture
- Stream API
- Optional types

## Adding Kiota to Our Weather App 🌤️

Let's add Kiota client generation to our React frontend from the Aspire series. We'll generate a TypeScript client from the OpenAPI spec we created with TypeSpec.

### Prerequisites 📦

From our previous posts, we already have:
- 🌤️ Weather API with TypeSpec contract
- 📄 Generated OpenAPI specification
- ⚛️ React TypeScript frontend
- 🔧 Node.js installed in our dev container

### Step 1: Install Kiota CLI 🛠️

Install Kiota as a .NET global tool:

```bash
dotnet tool install -g Microsoft.OpenApi.Kiota
```

> **Note:** This requires .NET 8.0 SDK to be installed. If you're working in a dev container or have .NET already installed for your backend development, this is the recommended approach.

Verify the installation:

```bash
kiota --version
```

### Step 2: Add Kiota Generation Script to package.json 📝

Navigate to your React app directory (`src/WeatherApp.Web`) and update `package.json` to add a script for generating the Kiota client:

```json
{
  "name": "weatherapp.web",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview",
    "generate-client": "kiota generate -l typescript -c WeatherApiClient -n client -d ../generated/openapi/openapi.yaml -o ./src/client"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "@microsoft/kiota-bundle": "^1.0.0-preview.99"
  },
  "devDependencies": {
    "@eslint/js": "^9.13.0",
    "@types/react": "^18.3.12",
    "@types/react-dom": "^18.3.1",
    "@vitejs/plugin-react": "^4.3.3",
    "eslint": "^9.13.0",
    "eslint-plugin-react-hooks": "^5.0.0",
    "eslint-plugin-react-refresh": "^0.4.14",
    "globals": "^15.11.0",
    "typescript": "~5.6.2",
    "typescript-eslint": "^8.11.0",
    "vite": "^5.4.10"
  }
}
```

**Let's break down the `generate-client` command:** 🔍

```bash
kiota generate \
  -l typescript \                    # Language: TypeScript
  -c WeatherApiClient \              # Client class name
  -n client \                        # Namespace/module name
  -d ../generated/openapi/openapi.yaml \ # OpenAPI spec path (from TypeSpec emitter)
  -o ./src/client                    # Output directory for generated files
```

**Parameters explained:**
- 🔤 `-l typescript`: Generate TypeScript client
- 🏷️ `-c WeatherApiClient`: Name of the main client class
- 📂 `-n client`: Namespace/module name for generated code (must conform to `^[\w][\w\._-]+`)
- 📄 `-d ../generated/openapi/openapi.yaml`: Path to OpenAPI spec (from TypeSpec's `emitter-output-dir` configuration)
- 📁 `-o ./src/client`: Where to write generated files

### Step 3: Install Kiota Dependencies 📦

The generated client will need Kiota's runtime libraries. Install the bundle package which includes all necessary dependencies:

```bash
# From WeatherApp.Web directory
npm install @microsoft/kiota-bundle
```

The `@microsoft/kiota-bundle` package includes:
- 🎯 **@microsoft/kiota-abstractions**: Core abstractions and interfaces
- 🌐 **@microsoft/kiota-http-fetchlibrary**: HTTP adapter using browser's fetch API
- 📋 **@microsoft/kiota-serialization-json**: JSON serialization/deserialization

This makes setup simple with just one dependency to manage.

### Step 4: Generate the Client 🤖

Now run the generation script:

```bash
npm run generate-client
```

This will create a `src/client/` directory with your generated TypeScript client:

```
src/client/
├── index.ts                      # Main exports
├── weatherApiClient.ts           # Main client class
├── models/                       # Generated model types
│   ├── weatherForecast.ts
│   └── index.ts
└── weather/                      # API endpoint methods
    ├── index.ts
    └── weatherRequestBuilder.ts
```

### Step 5: Configure the Client in Your React App 🔧

Create a client configuration file `src/services/weatherService.ts`:

```typescript
import { AnonymousAuthenticationProvider } from '@microsoft/kiota-abstractions';
import { FetchRequestAdapter } from '@microsoft/kiota-http-fetchlibrary';
import { createWeatherApiClient } from '../client/weatherApiClient.ts';

// Create an anonymous authentication provider (no auth required)
const authProvider = new AnonymousAuthenticationProvider();

// Create the HTTP adapter with the auth provider
const adapter = new FetchRequestAdapter(authProvider);

// Set the base URL for your API
// Uses '/api' to leverage Vite's proxy configuration for local development
adapter.baseUrl = '/api';

// Create and export the configured client
export const weatherApiClient = createWeatherApiClient(adapter);
```

> **Note:** We're using `AnonymousAuthenticationProvider` since our Weather API doesn't require authentication. For APIs that require authentication (OAuth, API keys, etc.), you would use a different authentication provider. We're also using `/api` as the base URL to take advantage of Vite's proxy configuration.

### Step 6: Use the Generated Client in React 💻

Now update your React component to use the Kiota-generated client instead of manual fetch calls:

**Before (Manual Fetch):**
```typescript
const [forecasts, setForecasts] = useState<WeatherForecast[]>([]);

useEffect(() => {
  fetch('/api/weatherforecast')
    .then(res => res.json())
    .then(data => setForecasts(data));
}, []);
```

**After (Kiota Client):**
```typescript
import { useState, useEffect } from 'react';
import { weatherApiClient } from './services/weatherService.ts';
import type { WeatherForecast } from './client/models/index.ts';

function App() {
  const [forecasts, setForecasts] = useState<WeatherForecast[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadForecasts = async () => {
      try {
        const data = await weatherApiClient.weatherforecast.get();
        if (data) {
          setForecasts(data);
          setLoading(false);
        }
      } catch (error) {
        console.error('Failed to load weather forecasts:', error);
        setError('Failed to load weather forecasts');
        setLoading(false);
      }
    };
    
    loadForecasts();
  }, []);

  if (loading) {
    return <div className="loading">Loading weather data...</div>;
  }

  if (error) {
    return (
      <div className="error">
        <h2>Error</h2>
        <p>{error}</p>
        <button onClick={() => window.location.reload()}>Retry</button>
      </div>
    );
  }

  return (
    <div className="weather-app">
      <h1>Weather Forecast</h1>
      {/* Render your weather forecast data */}
      <ul>
        {forecasts.map((forecast, index) => (
          <li key={index}>{/* forecast data */}</li>
        ))}
      </ul>
    </div>
  );
}
```

> **⚠️ Important Note about `additionalData`:** Depending on your OpenAPI schema and how Kiota generates the models, you may find that properties are stored in an `additionalData` dictionary rather than as direct properties on the model. If you encounter this, you can access the data like this:
> ```typescript
> const data = forecast.additionalData as any;
> const temperature = data?.TemperatureC;
> const summary = data?.Summary;
> ```
> This is a known behavior when the OpenAPI schema doesn't have a strict object definition or when using certain serialization settings. You can work around this by refining your TypeSpec schema to ensure properties are properly typed, or by creating a wrapper type for better type safety.

**Benefits of the Kiota approach:** ✅
- ✨ Full TypeScript IntelliSense for API methods and models
- 🔒 Type safety with generated `WeatherForecast` interface
- 🎯 Autocomplete for API endpoints (`weatherApiClient.weatherforecast.get()`)
- ⚠️ Compile-time errors if API contract changes
- 🧹 Cleaner, more maintainable code with proper error handling

## Workflow: From TypeSpec to Kiota Client 🔄

Here's the complete workflow we've built:

```
1. Define API Contract (TypeSpec)
   ↓
2. Generate OpenAPI Spec
   ↓
3. Generate Kiota Client
   ↓
4. Use in Application
```

**Commands summary:**

```bash
# 1. Generate OpenAPI from TypeSpec
cd WeatherApp.Typespec
tsp compile .

# 2. Generate Kiota client for React
cd ../src/WeatherApp.Web
npm run generate-client

# 3. Run your app with Aspire (requires Aspire CLI installed)
aspire run
```

> **Note:** `aspire run` starts the entire application including the backend API and frontend, orchestrated by .NET Aspire. This requires the [.NET Aspire workload](https://learn.microsoft.com/en-us/dotnet/aspire/fundamentals/setup-tooling) to be installed.

## When to Regenerate the Client 🔄

You should regenerate your Kiota client whenever:

1. 📝 **API Contract Changes**: TypeSpec models or operations are modified
2. 🔧 **OpenAPI Spec Updates**: New endpoints or parameters added
3. 🐛 **Bug Fixes**: Issues in the OpenAPI specification are corrected
4. ⬆️ **Breaking Changes**: Major version updates to your API

**Automation Tip:** 💡
Add the client generation to your CI/CD pipeline:

```yaml
# Azure DevOps example
- script: |
    npm run generate-client
  displayName: 'Generate Kiota Client'
  workingDirectory: src/WeatherApp.Web
```

## Advantages Over Manual Client Code ✨

### Type Safety
- ✅ Compiler catches API contract mismatches
- ✅ IntelliSense shows available operations
- ✅ Refactoring tools work correctly

### Consistency
- ✅ Same patterns across all API calls
- ✅ Uniform error handling
- ✅ Consistent authentication

### Maintainability
- ✅ One command to update client when API changes
- ✅ No manual JSON parsing code
- ✅ Less boilerplate

### Developer Experience
- ✅ Discover APIs through autocomplete
- ✅ Self-documenting code
- ✅ Faster development

## Multi-Language Support 🌍

While we used TypeScript for our React app, Kiota can generate clients for:

**Supported Languages:**
- 🟦 **TypeScript/JavaScript**: Browser and Node.js
- 🟪 **C#**: .NET applications
- ☕ **Java**: Spring Boot, Android
- 🐍 **Python**: Django, Flask, FastAPI
- 🔷 **Go**: Backend services
- 🐘 **PHP**: Laravel, WordPress
- 💎 **Ruby**: Rails applications

**Same OpenAPI spec, multiple clients:**

```bash
# Generate C# client for backend services
kiota generate -l csharp -c WeatherClient -n MyApp.Clients -d openapi.yaml -o ./clients/csharp

# Generate Python client for data science teams
kiota generate -l python -c WeatherClient -n weather_client -d openapi.yaml -o ./clients/python

# Generate Java client for Android app
kiota generate -l java -c WeatherClient -n com.myapp.clients -d openapi.yaml -o ./clients/java
```

## Best Practices 🎯

### 1. Version Your Generated Code 📚

**Option A: Check in generated code**
- ✅ Pros: Reviewable diffs, no build dependency
- ❌ Cons: Merge conflicts, larger repo

**Option B: Generate on build**
- ✅ Pros: Always up-to-date, smaller repo
- ❌ Cons: Build-time dependency, potential breakage

**Recommendation:** Check in generated code initially, regenerate when spec changes.

### 2. Separate Client Configuration 🔧

Keep API client setup separate from business logic:

```typescript
// src/services/apiClient.ts - Configuration
export const weatherApiClient = createWeatherApiClient(adapter);

// src/hooks/useWeather.ts - Business logic
export function useWeather() {
  return useQuery({
    queryKey: ['weather'],
    queryFn: () => weatherApiClient.weather.get()
  });
}
```

### 3. Add Error Handling 🛡️

Wrap client calls with proper error handling:

```typescript
async function getForecasts() {
  try {
    const forecasts = await weatherApiClient.weather.get();
    return { data: forecasts, error: null };
  } catch (error) {
    console.error('API Error:', error);
    return { data: null, error: error as Error };
  }
}
```

### 4. Use Environment Variables 🔐

Don't hardcode API URLs:

```typescript
// .env.development
VITE_API_URL=http://localhost:5000

// .env.production  
VITE_API_URL=https://api.production.com

// apiClient.ts
adapter.baseUrl = import.meta.env.VITE_API_URL;
```

## Troubleshooting 🔧

### Issue: "Module not found" after generation

**Solution:** Make sure the Kiota bundle package is installed:
```bash
npm install @microsoft/kiota-bundle
```

### Issue: Generated code has type errors

**Solution:** Ensure TypeScript version compatibility:
```bash
npm install -D typescript@latest
```

### Issue: API calls fail with CORS errors

**Solution:** Configure CORS in your API (already done in our Aspire setup):
```csharp
builder.Services.AddCors(options =>
````

### Issue: "Module not found" after generation

**Solution:** Make sure the Kiota bundle package is installed:
```bash
npm install @microsoft/kiota-bundle
```

### Issue: Generated code has type errors

**Solution:** Ensure TypeScript version compatibility:
```bash
npm install -D typescript@latest
```

### Issue: API calls fail with CORS errors

**Solution:** Configure CORS in your API (already done in our Aspire setup):
```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

## What's Next 🔮

In upcoming posts, we'll explore:
- 🏗️ Terraform patterns for Azure infrastructure
- 📦 Creating custom .NET templates
- 🔄 Azure DevOps pipeline setup
- 🎭 Service virtualization with WireMock

Each tool builds on the previous ones, creating a cohesive modern development platform.

## Conclusion 🎉

Kiota completes our contract-first development workflow:

1. ✅ TypeSpec defines the API contract
2. ✅ OpenAPI spec is generated
3. ✅ Kiota creates type-safe clients
4. ✅ Frontend/backend teams work in parallel

The result? **Faster development, fewer bugs, and happier developers.**

No more manual JSON parsing. No more keeping TypeScript interfaces in sync with C# models. No more "the API changed and broke everything" surprises.

With Kiota, your API contract is your source of truth, and changes propagate automatically to all consumers.

**Try it out:** Add Kiota to your next API project and experience the difference contract-first development makes!

---

**Resources:** 📚

- [Kiota Documentation](https://learn.microsoft.com/en-us/openapi/kiota/overview)
- [Kiota GitHub Repository](https://github.com/microsoft/kiota)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Our GitHub Repository](https://github.com/two4suited/blog-platform-aspire)
