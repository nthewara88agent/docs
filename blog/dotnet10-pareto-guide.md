# The .NET 10 Pareto Guide: The 20% That Gives You 80%

*Published: March 1, 2026*

## Introduction

.NET 10 landed in November 2025, and the feature list is enormous. If you're coming from .NET Framework, another language, or just want to get productive fast — you don't need to learn everything. You need to learn the *right* things.

This guide applies the Pareto principle to .NET 10: the 20% of concepts that give you 80% of what you need to build real applications. No fluff, just the stuff that matters.

## 1. Minimal APIs + Built-in OpenAPI

Gone are the days of heavyweight controllers and Swashbuckle configuration. Minimal APIs are the default way to build HTTP APIs in .NET 10, and OpenAPI support is now built in.

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi();

var app = builder.Build();
app.MapOpenApi();

app.MapGet("/products/{id}", (int id) => new { Id = id, Name = "Widget" });

app.MapPost("/products", (Product product) => Results.Created($"/products/{product.Id}", product));

app.Run();

record Product(int Id, string Name, decimal Price);
```

That's a fully functional API with an OpenAPI 3.1 spec generated at `/openapi/v1.json`. No Swagger packages, no XML comments config, no startup ceremony.

**Why this matters:** Every web project you build will start here. Master this pattern and you're immediately productive.

## 2. C# File-Based Applications

This is genuinely revolutionary. .NET 10 lets you run a single `.cs` file without a `.csproj`, `Program.cs` ceremony, or solution file:

```csharp
// app.cs
#:package Microsoft.EntityFrameworkCore.Sqlite@10.*

using Microsoft.EntityFrameworkCore;

var db = new MyDb();
db.Database.EnsureCreated();
db.Tasks.Add(new Todo("Learn .NET 10"));
db.SaveChanges();

foreach (var task in db.Tasks)
    Console.WriteLine(task);

class MyDb : DbContext
{
    public DbSet<Todo> Tasks => Set<Todo>();
    protected override void OnConfiguring(DbContextOptionsBuilder o)
        => o.UseSqlite("Data Source=tasks.db");
}

record Todo(string Title)
{
    public int Id { get; init; }
}
```

Run it with:

```bash
dotnet run app.cs
```

No project file. NuGet packages declared inline with `#:package`. This is a game-changer for scripting, prototyping, and learning.

## 3. Dependency Injection & the Middleware Pipeline

This is the single most important concept in ASP.NET Core. Every request flows through a pipeline of middleware, and every service is resolved through dependency injection (DI). If you understand this, you understand the framework.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register services (DI container)
builder.Services.AddSingleton<ICache, RedisCache>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddHttpClient<IGitHubClient, GitHubClient>();

var app = builder.Build();

// Middleware pipeline (order matters!)
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();

// Inject services directly into endpoints
app.MapGet("/orders/{id}", async (int id, IOrderService orders) =>
    await orders.GetAsync(id) is { } order
        ? Results.Ok(order)
        : Results.NotFound());

app.Run();
```

**The three lifetimes you need to know:**

| Lifetime | Behaviour | Use for |
|-----------|-----------|---------|
| `Singleton` | One instance for the app's lifetime | Caches, configuration |
| `Scoped` | One instance per HTTP request | Database contexts, unit-of-work |
| `Transient` | New instance every time | Lightweight, stateless services |

**Why this matters:** Every single ASP.NET Core feature — authentication, EF Core, HTTP clients, health checks — plugs into DI. Learn it once, use it everywhere.

## 4. Entity Framework Core 10

EF Core is the default data access layer. The code-first workflow means you define C# classes and EF generates your database schema.

```csharp
// Define your model
public class Blog
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Url { get; set; } = "";
    public List<Post> Posts { get; set; } = [];
}

public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
    public int BlogId { get; set; }
    public Blog Blog { get; set; } = null!;
}

// Define your context
public class AppDb(DbContextOptions<AppDb> options) : DbContext(options)
{
    public DbSet<Blog> Blogs => Set<Blog>();
    public DbSet<Post> Posts => Set<Post>();
}
```

Register it in DI and use migrations:

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

Query with LINQ:

```csharp
app.MapGet("/blogs", async (AppDb db) =>
    await db.Blogs
        .Where(b => b.Posts.Count > 0)
        .OrderByDescending(b => b.Posts.Count)
        .Select(b => new { b.Title, PostCount = b.Posts.Count })
        .ToListAsync());
```

**Why this matters:** Almost every app needs a database. EF Core handles the heavy lifting so you can focus on your domain.

## 5. Minimal API Validation

.NET 10 adds built-in validation to minimal APIs. No more installing FluentValidation for basic scenarios:

```csharp
using System.ComponentModel.DataAnnotations;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddValidation();

var app = builder.Build();

app.MapPost("/products", (Product product) =>
    Results.Created($"/products/{product.Id}", product));

app.Run();

record Product(
    int Id,
    [property: Required, StringLength(100)] string Name,
    [property: Range(0.01, 10000)] decimal Price
);
```

Validation errors automatically return a `400 Bad Request` with a structured problem details response. The framework handles it — you just annotate your types.

## 6. Blazor: Full-Stack C#

Blazor lets you build interactive web UIs in C# instead of JavaScript. .NET 10 brings persistent component state across enhanced navigations and improved render modes.

```csharp
// Components/Pages/Counter.razor
@page "/counter"
@rendermode InteractiveServer

<h1>Counter</h1>
<p>Current count: @currentCount</p>
<button @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;
    private void IncrementCount() => currentCount++;
}
```

The three render modes you need to know:

- **Static SSR** — Server-rendered HTML, no interactivity (fastest)
- **Interactive Server** — Real-time via SignalR WebSocket
- **Interactive WebAssembly** — Runs .NET in the browser, no server needed after download

**Why this matters:** If you're a C# developer, Blazor means full-stack without context-switching to a JavaScript framework.

## 7. Native AOT

Ahead-of-Time compilation produces a self-contained native binary. No JIT, no runtime installation needed. Startup in milliseconds instead of seconds.

```csharp
var builder = WebApplication.CreateSlimBuilder(args);
var app = builder.Build();

app.MapGet("/", () => "Hello, AOT!");

app.Run();
```

```xml
<!-- In .csproj -->
<PropertyGroup>
    <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -c Release
# Output: a single native binary, ~10-15 MB
```

**When to use it:**
- Container deployments (smaller images, faster cold starts)
- Serverless functions (Azure Functions, AWS Lambda)
- CLI tools

**When to skip it:**
- You need reflection-heavy libraries
- Development inner loop (JIT is faster to iterate with)

## 8. Configuration & Options Pattern

.NET 10 apps pull configuration from multiple sources, merged in order of priority:

```json
// appsettings.json
{
  "Database": {
    "ConnectionString": "Server=localhost;Database=myapp",
    "MaxRetries": 3
  },
  "Features": {
    "EnableBetaUI": false
  }
}
```

Bind configuration to strongly-typed classes with the Options pattern:

```csharp
public class DatabaseOptions
{
    public string ConnectionString { get; set; } = "";
    public int MaxRetries { get; set; } = 3;
}

// Registration
builder.Services.Configure<DatabaseOptions>(
    builder.Configuration.GetSection("Database"));

// Usage via DI
app.MapGet("/health", (IOptions<DatabaseOptions> opts) =>
    new { Database = opts.Value.ConnectionString != "" ? "configured" : "missing" });
```

**Configuration priority (highest wins):**
1. Command-line arguments
2. Environment variables
3. User secrets (development only)
4. `appsettings.{Environment}.json`
5. `appsettings.json`

**Why this matters:** Every real app needs configuration. The Options pattern gives you compile-time safety over string-based config access.

## 9. Authentication & Authorisation

.NET 10 provides a complete identity system out of the box:

```csharp
var builder = WebApplication.CreateBuilder(args);

// JWT Bearer auth
builder.Services.AddAuthentication()
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";
        options.Audience = "api://my-app";
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();

// Protect endpoints
app.MapGet("/admin/dashboard", () => "Secret stuff")
    .RequireAuthorization("AdminPolicy");

app.MapGet("/profile", (ClaimsPrincipal user) =>
    new { Name = user.Identity?.Name })
    .RequireAuthorization();

app.Run();
```

For full user management (registration, login, password reset), scaffold the Identity system:

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

**Key patterns:**
- `[Authorize]` or `.RequireAuthorization()` to protect endpoints
- Claims-based identity for user information
- Policy-based authorisation for complex rules
- JWT for API authentication, cookies for web apps

## 10. Performance & Diagnostics

.NET 10 makes observability a first-class citizen. Health checks and OpenTelemetry are built in:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Health checks
builder.Services.AddHealthChecks()
    .AddCheck("database", () =>
    {
        // Check your DB connection
        return HealthCheckResult.Healthy();
    })
    .AddCheck<RedisHealthCheck>("cache");

// OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithMetrics(m => m.AddAspNetCoreInstrumentation())
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());

var app = builder.Build();
app.MapHealthChecks("/healthz");

app.Run();
```

**Why this matters:** Production apps need monitoring. Health checks give you liveness/readiness probes for Kubernetes. OpenTelemetry gives you distributed tracing across services. Both are essential for anything beyond a toy project.

## Putting It All Together

Here's a complete .NET 10 API that uses all ten concepts:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// DI + Configuration (#3, #8)
builder.Services.Configure<AppOptions>(builder.Configuration.GetSection("App"));
builder.Services.AddDbContext<AppDb>(o =>
    o.UseSqlite(builder.Configuration.GetConnectionString("Default")));

// OpenAPI (#1)
builder.Services.AddOpenApi();
builder.Services.AddValidation(); // (#5)

// Auth (#9)
builder.Services.AddAuthentication().AddJwtBearer();
builder.Services.AddAuthorization();

// Health checks (#10)
builder.Services.AddHealthChecks();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.MapOpenApi();
app.MapHealthChecks("/healthz");

// Minimal API endpoints (#1)
app.MapGet("/products", async (AppDb db) =>
    await db.Products.ToListAsync());

app.MapPost("/products", async (Product product, AppDb db) =>
{
    db.Products.Add(product);
    await db.SaveChangesAsync();
    return Results.Created($"/products/{product.Id}", product);
}).RequireAuthorization();

app.Run();

// EF Core model (#4)
record Product(
    int Id,
    [property: Required] string Name,  // Validation (#5)
    [property: Range(0.01, 10000)] decimal Price
);

class AppDb(DbContextOptions<AppDb> options) : DbContext(options)
{
    public DbSet<Product> Products => Set<Product>();
}

class AppOptions
{
    public string AppName { get; set; } = "";
}
```

## What's Deliberately Not Here

This guide skips plenty of .NET 10 features: gRPC, SignalR, background services, Aspire, distributed caching, rate limiting, output caching, and more. Those are important — but they're the *next* 20%, not the first.

Master these ten concepts and you'll be able to build, ship, and maintain real .NET 10 applications. Everything else builds on top of this foundation.

## Resources

- [.NET 10 Documentation](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/overview)
- [ASP.NET Core 10 What's New](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-10.0)
- [Entity Framework Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)
- [C# File-Based Apps](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-14#file-based-programs)
