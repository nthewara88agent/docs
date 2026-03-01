# From .NET Framework to .NET 10: The Migration Conversation Guide

*Published: March 1, 2026*

You've spent years building applications on .NET Framework 4.x — ASP.NET MVC, Web Forms, WCF, the works. Now your customers are asking about .NET 10, and you need to have credible conversations about what migration actually involves.

This isn't a step-by-step migration guide. It's the Pareto version: the 20% of knowledge that gives you 80% coverage in customer conversations. After reading this, you'll understand what changed, what breaks, what's better, and how to advise customers on the "should we migrate?" question.

---

## 1. The Platform Shift: Understanding What Happened

The single most important thing to understand: **.NET Framework 4.8.1 is the last version of .NET Framework. Ever.** It will receive security patches, but no new features. The future is .NET (without "Framework").

Here's the naming journey that confuses everyone:

| Year | Release | What It Was |
|------|---------|-------------|
| 2002–2019 | .NET Framework 1.0 → 4.8 | Windows-only, closed source, ships with Windows |
| 2016 | .NET Core 1.0 | Cross-platform rewrite, open source |
| 2019 | .NET Core 3.1 | Feature parity push, LTS |
| 2020 | .NET 5 | Dropped "Core" from the name (skipped 4 to avoid confusion with Framework 4.x) |
| 2021–2024 | .NET 6 / 7 / 8 / 9 | Annual releases, even = LTS, odd = STS |
| 2025 | **.NET 10** | Current LTS release, the migration target |

**The customer talking point:** .NET Framework isn't going away tomorrow — it ships with Windows and gets security patches. But it's a dead end. No new features, no performance improvements, shrinking NuGet package support, and an increasingly difficult hiring market for Framework-only developers.

### What Changed Fundamentally

| Aspect | .NET Framework 4.x | .NET 10 |
|--------|-------------------|---------|
| Platform | Windows only | Windows, Linux, macOS |
| Source | Closed source | Open source (MIT licence) |
| Deployment | GAC, machine-wide install | App-local, self-contained, containers |
| Release cadence | Years between releases | Annual (Nov each year) |
| Performance | Good | 2–10x faster (depending on workload) |
| Runtime | CLR (ships with Windows) | CoreCLR (ships with your app) |

---

## 2. Project System: From XML Walls to Simplicity

If you've worked with .NET Framework `.csproj` files, you know the pain — hundreds of lines of XML, `ProjectTypeGuids`, every file explicitly listed. The SDK-style project format is dramatically simpler.

**Framework 4.x `.csproj` (abbreviated — the real ones were worse):**

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" DefaultTargets="Build"
         xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    <ProjectGuid>{GUID-HERE}</ProjectGuid>
    <OutputType>Library</OutputType>
    <RootNamespace>MyApp</RootNamespace>
    <AssemblyName>MyApp</AssemblyName>
    <TargetFrameworkVersion>v4.8</TargetFrameworkVersion>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="System" />
    <Reference Include="System.Core" />
    <Reference Include="System.Web" />
    <!-- 50+ more references... -->
  </ItemGroup>
  <ItemGroup>
    <Compile Include="Controllers\HomeController.cs" />
    <Compile Include="Models\Product.cs" />
    <!-- Every. Single. File. Listed. -->
  </ItemGroup>
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```

**.NET 10 SDK-style `.csproj`:**

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
  </ItemGroup>
</Project>
```

That's it. Files are included by convention (wildcard globbing). No GUIDs. No explicit system references. MSBuild still powers it under the hood, but you rarely need to think about it.

**Package management also changed:** `packages.config` is gone. NuGet references are `<PackageReference>` elements directly in the `.csproj`. No more `packages/` folder committed to source control.

---

## 3. Hosting Model: The Biggest Architectural Change

This is where Framework developers have the steepest learning curve. The entire hosting model is different.

**Framework 4.x:**
- IIS is your web server
- `System.Web.dll` is the runtime (tightly coupled to IIS)
- `Global.asax` handles application lifecycle events
- `web.config` configures everything
- HTTP modules and HTTP handlers form the pipeline

**.NET 10:**
- **Kestrel** is the built-in web server (IIS becomes a reverse proxy)
- Middleware pipeline replaces HTTP modules/handlers
- `Program.cs` is your entry point (it's just a console app)
- `appsettings.json` replaces `web.config` for app config
- `web.config` still exists but only for IIS integration settings

**Framework 4.x — `Global.asax`:**

```csharp
public class MvcApplication : System.Web.HttpApplication
{
    protected void Application_Start()
    {
        AreaRegistration.RegisterAllAreas();
        FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
        RouteConfig.RegisterRoutes(RouteTable.Routes);
        BundleConfig.RegisterBundles(BundleTable.Bundles);
    }
}
```

**.NET 10 — `Program.cs` (minimal hosting):**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

var app = builder.Build();

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorisation();
app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

No `Startup.cs`, no `Global.asax`, no `Application_Start`. The entire application bootstrap is a top-level program. Earlier versions of .NET Core used a `Startup.cs` with `ConfigureServices()` and `Configure()` methods — that still works but minimal hosting in `Program.cs` is now the standard.

**The middleware pipeline** is the key concept. Instead of HTTP modules hooking into fixed IIS pipeline events, you compose middleware in order:

```csharp
app.UseExceptionHandler("/Error");  // Runs first (outermost)
app.UseHttpsRedirection();
app.UseStaticFiles();               // Short-circuits for static files
app.UseRouting();
app.UseAuthentication();            // Who are you?
app.UseAuthorisation();             // Are you allowed?
app.MapControllers();               // Endpoint dispatch
```

Order matters. Authentication must come before authorisation. Static files should come early to skip unnecessary processing. This is explicit and visible, unlike the Framework pipeline where module ordering was buried in `web.config`.

---

## 4. Dependency Injection: Now It's Built In

This is arguably the **#1 paradigm shift** for Framework developers. In Framework 4.x, if you wanted DI, you installed a third-party container — Ninject, Autofac, Unity, StructureMap. Many Framework apps have no DI at all, just `new` everywhere.

In .NET 10, DI is built into the framework. It's not optional — the entire framework is designed around it.

**Framework 4.x (with Ninject):**

```csharp
// NinjectWebCommon.cs
kernel.Bind<IProductRepository>().To<SqlProductRepository>();
kernel.Bind<IEmailService>().To<SmtpEmailService>().InSingletonScope();
```

**.NET 10 (built-in DI):**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IProductRepository, SqlProductRepository>();
builder.Services.AddSingleton<IEmailService, SmtpEmailService>();
builder.Services.AddTransient<IReportGenerator, PdfReportGenerator>();
```

The three lifetimes you need to know:

| Lifetime | Behaviour | Use For |
|----------|-----------|---------|
| **Transient** | New instance every time | Lightweight, stateless services |
| **Scoped** | One instance per HTTP request | DbContext, unit of work |
| **Singleton** | One instance for the app lifetime | Caches, configuration, HTTP clients |

Constructor injection just works:

```csharp
public class ProductController : Controller
{
    private readonly IProductRepository _repository;

    public ProductController(IProductRepository repository)
    {
        _repository = repository; // Injected automatically
    }
}
```

**Customer talking point:** If the existing Framework codebase uses DI already, migration is smoother. If it's full of `new` and static helpers, there's refactoring work to do — but it's refactoring that should happen anyway.

---

## 5. Configuration: From XML Hell to Structured Config

**Framework 4.x:**

```xml
<!-- web.config -->
<configuration>
  <appSettings>
    <add key="ApiUrl" value="https://api.example.com" />
    <add key="MaxRetries" value="3" />
  </appSettings>
  <connectionStrings>
    <add name="DefaultConnection"
         connectionString="Server=.;Database=MyDb;Integrated Security=true"
         providerName="System.Data.SqlClient" />
  </connectionStrings>
</configuration>
```

```csharp
// Accessing config
var apiUrl = ConfigurationManager.AppSettings["ApiUrl"];  // string, no type safety
var connStr = ConfigurationManager.ConnectionStrings["DefaultConnection"].ConnectionString;
```

**.NET 10:**

```json
// appsettings.json
{
  "ApiSettings": {
    "Url": "https://api.example.com",
    "MaxRetries": 3,
    "TimeoutSeconds": 30
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=MyDb;Trusted_Connection=true"
  }
}
```

```csharp
// Strongly-typed configuration with IOptions<T>
public class ApiSettings
{
    public string Url { get; set; }
    public int MaxRetries { get; set; }
    public int TimeoutSeconds { get; set; }
}

// In Program.cs
builder.Services.Configure<ApiSettings>(builder.Configuration.GetSection("ApiSettings"));

// In your service (injected)
public class ApiClient
{
    private readonly ApiSettings _settings;

    public ApiClient(IOptions<ApiSettings> options)
    {
        _settings = options.Value;  // Strongly typed!
    }
}
```

The configuration system layers multiple sources with override precedence:
1. `appsettings.json`
2. `appsettings.{Environment}.json` (e.g., `appsettings.Production.json`)
3. Environment variables
4. User secrets (development only)
5. Command-line arguments

No more Web.config transforms for different environments. Set an environment variable and the right config loads automatically.

---

## 6. Entity Framework: EF6 → EF Core 10

If your customers use Entity Framework 6, this is a significant migration area.

| Feature | EF6 | EF Core 10 |
|---------|-----|------------|
| EDMX designer | ✅ Supported | ❌ Gone — code-first only |
| Lazy loading | ✅ On by default | Opt-in (requires proxy package) |
| Database-first | ✅ Full support | ⚠️ Scaffolding exists but code-first preferred |
| LINQ translation | Client-side fallback | Strict server-side (throws if it can't translate) |
| Migrations CLI | `Enable-Migrations` / `Update-Database` (PMC) | `dotnet ef migrations add` / `dotnet ef database update` |
| `ObjectContext` | ✅ | ❌ Only `DbContext` |
| Spatial data | Limited | ✅ Full NetTopologySuite support |
| Performance | Good | Significantly better |

**The LINQ translation change is the biggest gotcha.** EF6 would silently evaluate queries client-side if it couldn't translate to SQL. EF Core throws an exception. This means queries that "worked" in EF6 may fail in EF Core — they were always inefficient, but now they're explicit failures.

**Framework 4.x (EF6):**

```csharp
// This worked in EF6 (evaluated client-side, slow)
var results = context.Products
    .Where(p => p.Name.Contains(searchTerm, StringComparison.OrdinalIgnoreCase))
    .ToList();
```

**.NET 10 (EF Core 10):**

```csharp
// EF Core translates to SQL properly
var results = await context.Products
    .Where(p => EF.Functions.Like(p.Name, $"%{searchTerm}%"))
    .ToListAsync();
```

---

## 7. ASP.NET MVC → ASP.NET Core MVC / Minimal APIs

Good news: **controllers still exist and work similarly.** This is the easiest migration path for MVC apps.

**What's familiar:**
- `Controller` base class, `IActionResult`, `ViewResult`, `JsonResult`
- Razor views (`.cshtml`) with `@model`, `@foreach`, etc.
- Model binding, validation attributes (`[Required]`, `[StringLength]`)
- `ViewBag`, `ViewData` (still there, though `IOptions<T>` is preferred)
- `[Route]`, `[HttpGet]`, `[HttpPost]` attributes

**What changed:**

| Framework 4.x | .NET 10 |
|---------------|---------|
| HTML Helpers: `@Html.TextBoxFor(m => m.Name)` | Tag Helpers: `<input asp-for="Name" />` |
| `@Html.ActionLink("Home", "Index", "Home")` | `<a asp-controller="Home" asp-action="Index">Home</a>` |
| `@Html.AntiForgeryToken()` | `<form method="post">` (auto-included) |
| Bundling via `BundleConfig.cs` | No built-in bundler — use npm/webpack or `WebOptimizer` |
| `FilterConfig.RegisterGlobalFilters()` | `builder.Services.AddControllers(o => o.Filters.Add(...))` |

**Minimal APIs** are the new default for API-only projects. No controllers, no ceremony:

```csharp
app.MapGet("/api/products", async (AppDbContext db) =>
    await db.Products.ToListAsync());

app.MapGet("/api/products/{id}", async (int id, AppDbContext db) =>
    await db.Products.FindAsync(id) is Product p
        ? Results.Ok(p)
        : Results.NotFound());

app.MapPost("/api/products", async (Product product, AppDbContext db) =>
{
    db.Products.Add(product);
    await db.SaveChangesAsync();
    return Results.Created($"/api/products/{product.Id}", product);
});
```

**Customer talking point:** If they have an MVC app with Razor views, controllers migrate fairly directly. If they have a Web API, they can keep controllers or move to Minimal APIs. The migration isn't a complete rewrite for MVC apps.

---

## 8. Authentication: Same Attributes, Different Plumbing

The `[Authorize]` attribute still works. But everything underneath is completely different.

**Framework 4.x:**
- Forms Authentication (`FormsAuthentication.SetAuthCookie()`)
- Windows Authentication (IIS integrated)
- `System.Web.Security.Membership` provider
- `Thread.CurrentPrincipal` / `HttpContext.Current.User`

**.NET 10:**
- ASP.NET Core Identity (replaces Membership)
- Cookie authentication (replaces Forms Auth)
- JWT Bearer tokens for APIs
- OAuth 2.0 / OpenID Connect first-class support
- `HttpContext.User` (no more `HttpContext.Current` — it's injected)

```csharp
// Program.cs - configuring auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant}/v2.0";
        options.Audience = "api://my-api";
    });

builder.Services.AddAuthorisation(options =>
{
    options.AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"));
});
```

```csharp
// Controller - same attribute, different engine
[Authorize(Policy = "AdminOnly")]
public class AdminController : Controller
{
    public IActionResult Dashboard() => View();
}
```

**The big gotcha:** `HttpContext.Current` is **gone**. In Framework 4.x, you could access the current HTTP context from anywhere — services, helpers, static classes. In .NET 10, `HttpContext` is only available through DI (`IHttpContextAccessor`) or directly in controllers/middleware. This single change can require significant refactoring in codebases that use `HttpContext.Current` liberally.

---

## 9. What Doesn't Migrate Easily

This is the section that matters most in customer conversations. Be honest about what's hard.

| Technology | Migration Path | Effort |
|-----------|---------------|--------|
| **WCF (server-side)** | Use **gRPC** or **CoreWCF** (community port) | 🔴 High — gRPC requires protocol changes; CoreWCF has limitations |
| **Web Forms** | **No equivalent.** Rewrite in Blazor, Razor Pages, or MVC | 🔴 High — complete rewrite |
| **ASMX web services** | Rewrite as Web API or Minimal API | 🔴 High |
| **System.Web dependencies** | Find alternatives or rewrite | 🟡 Medium |
| **`HttpContext.Current`** | Replace with `IHttpContextAccessor` injection | 🟡 Medium (widespread refactoring) |
| **GAC assemblies** | Bundle with app or use NuGet packages | 🟡 Medium |
| **`Thread.CurrentPrincipal`** | Use `HttpContext.User` via DI | 🟢 Low–Medium |
| **`ConfigurationManager`** | Replace with `IConfiguration` / `IOptions<T>` | 🟢 Low |
| **ASP.NET MVC 5** | Migrate to ASP.NET Core MVC | 🟢 Low–Medium (mostly namespace changes) |
| **Entity Framework 6** | Migrate to EF Core | 🟡 Medium (LINQ query changes, no EDMX) |

**The hard truth for customers:** If the application is heavily Web Forms or WCF-based, migration is essentially a rewrite. If it's MVC + Web API + EF6, it's a more manageable migration with clear paths.

---

## 10. Migration Tooling & Strategy

### .NET Upgrade Assistant

Microsoft's official migration tool. It handles:
- Converting `.csproj` to SDK-style
- Updating NuGet packages
- Replacing `System.Web` references
- Updating `Startup` code patterns

```bash
# Install
dotnet tool install -g upgrade-assistant

# Analyse a project
upgrade-assistant analyze MyApp.sln

# Upgrade interactively
upgrade-assistant upgrade MyApp.sln
```

It won't do everything, but it handles the mechanical parts and gives you a clear list of what needs manual attention.

### Incremental Migration with YARP

For large applications, a big-bang migration is risky. The **strangler fig pattern** using YARP (Yet Another Reverse Proxy) lets you migrate incrementally:

1. Stand up a new .NET 10 app with YARP
2. YARP proxies all requests to the existing Framework app
3. Migrate one controller/endpoint at a time to .NET 10
4. YARP routes migrated endpoints to the new app, everything else to the old one
5. Eventually, all traffic goes to the new app

This is Microsoft's recommended approach for large-scale migrations.

### The Decision Framework

| Scenario | Recommendation |
|----------|---------------|
| Small MVC app, < 50 controllers | Upgrade Assistant + manual fixes |
| Large MVC/API app | YARP strangler fig, incremental |
| Web Forms app | Rewrite (consider Blazor) |
| WCF service | Evaluate CoreWCF vs gRPC rewrite |
| App that works fine, no changes planned | Leave it on Framework (seriously) |

---

## 11. Deployment: New Options, New World

**Framework 4.x deployment:**
- Web Deploy / MSDeploy
- xcopy to IIS folder
- Requires .NET Framework installed on server
- Windows Server only
- IIS required

**.NET 10 deployment:**

```bash
# Self-contained — no .NET runtime needed on the server
dotnet publish -c Release --self-contained -r linux-x64

# Framework-dependent — smaller, requires .NET runtime installed
dotnet publish -c Release

# Docker container
docker build -t myapp .
docker run -p 8080:8080 myapp
```

**Dockerfile for a .NET 10 app:**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Key deployment options in .NET 10:**

| Option | When to Use |
|--------|-------------|
| **IIS (reverse proxy to Kestrel)** | Familiar path, Windows Server environments |
| **Docker containers** | Cloud-native, Azure Container Apps, AKS |
| **Self-contained executable** | Single-file deployment, no runtime dependency |
| **Azure App Service (Linux)** | PaaS, cheapest Linux plans available |
| **systemd service (Linux)** | Run as a daemon on Linux VMs |

**Customer talking point:** Moving to .NET 10 opens up Linux hosting, which typically costs 30-50% less than Windows Server licensing. Container deployments enable modern CI/CD practices and cloud-native architectures.

---

## 12. The Customer Conversation: Why (and Why Not) to Migrate

### Why Migrate

| Benefit | Detail |
|---------|--------|
| **Performance** | ASP.NET Core is consistently 2-10x faster than Framework in benchmarks. TechEmpower benchmarks show it competing with Go and Rust. |
| **Cross-platform** | Deploy on Linux, run in containers, develop on macOS. Removes Windows Server licence costs. |
| **Active development** | New features, performance improvements, security patches every release. Framework is frozen. |
| **Container-ready** | First-class Docker support, small container images (~80 MB base). |
| **Talent pool** | New .NET developers learn .NET (Core), not Framework. Hiring gets harder every year. |
| **NuGet ecosystem** | Many packages are dropping Framework support. You'll get stuck on old versions. |
| **Cloud-native** | Built-in support for health checks, OpenTelemetry, gRPC, minimal APIs — the patterns cloud providers expect. |
| **Security** | Actively patched. Framework gets critical fixes only. |

### Why NOT Migrate

| Reason | Detail |
|--------|--------|
| **Stable app, no changes needed** | If it works, it's not exposed to the internet, and no one touches the code — leave it. Migration has cost and risk. |
| **Heavy Web Forms / WCF** | Migration is effectively a rewrite. The ROI calculation is different. |
| **Timeline constraints** | A rushed migration is worse than a planned one. |
| **Regulatory / compliance** | Some environments require extensive re-certification after platform changes. Factor this into timelines. |

### ROI Talking Points

- **Infrastructure savings:** Linux hosting, smaller container footprint, better performance = fewer instances needed
- **Developer productivity:** Modern tooling, hot reload, faster build times, better debugging
- **Reduced risk:** Active security patches, growing ecosystem support
- **Hiring:** The developer market is moving to modern .NET. Framework-only skills are increasingly niche.
- **Total cost of delay:** Every year you wait, the gap widens and migration gets harder as packages drop Framework support

---

## Customer FAQ

**Q: Can we run .NET Framework and .NET 10 side by side?**
A: Yes. They use separate runtimes and can coexist on the same server. This is exactly what the YARP strangler fig approach relies on.

**Q: Is .NET 10 an LTS release?**
A: Yes. Even-numbered releases (.NET 6, 8, 10) are Long-Term Support with 3 years of support. Odd-numbered releases (.NET 7, 9) are Standard-Term Support with 18 months.

**Q: Do we need to rewrite everything?**
A: It depends on the technology. ASP.NET MVC apps can often be migrated with moderate effort using the Upgrade Assistant. Web Forms and WCF services require rewrites. Class libraries with no `System.Web` dependencies are often the easiest — just retarget.

**Q: What about our stored procedures and database?**
A: The database doesn't change. SQL Server, PostgreSQL, whatever you're using stays the same. Only the data access layer (EF6 → EF Core) needs updating.

**Q: Can we still use IIS?**
A: Yes. IIS acts as a reverse proxy in front of Kestrel. The IIS integration works well and is a valid production deployment model. You just don't depend on IIS the way Framework does.

**Q: What about Windows Authentication?**
A: It's supported in ASP.NET Core via the Negotiate middleware. Works with IIS or Kestrel. The configuration is different but the functionality is there.

**Q: How long does migration typically take?**
A: A rough guide — small MVC app (< 20 controllers): 2-4 weeks. Medium MVC + API app: 1-3 months. Large enterprise app with mixed technologies: 6-18 months (incremental). Web Forms rewrite: treat it as a new project.

**Q: What if we use third-party libraries that only support .NET Framework?**
A: Check NuGet for .NET Standard 2.0 or .NET 10 versions. Many libraries have been ported. For those that haven't, you may need to find alternatives or use the Windows Compatibility Pack (`Microsoft.Windows.Compatibility`) which provides many Framework APIs on .NET 10.

**Q: Should we go straight to .NET 10 or step through intermediate versions?**
A: Go straight to .NET 10. There's no benefit to migrating to .NET Core 3.1 or .NET 6 first. The Upgrade Assistant targets the latest version.

**Q: What about our CI/CD pipelines?**
A: You'll need the .NET 10 SDK on your build agents. Azure DevOps and GitHub Actions both have .NET 10 tasks/actions. The build command changes from `msbuild MyApp.sln` to `dotnet build MyApp.sln` (though MSBuild still works).

---

## Summary: Your Conversation Cheat Sheet

When sitting with a customer, keep these key messages in mind:

1. **.NET Framework 4.8 is end of line.** Not end of life, but no new features, ever.
2. **MVC apps migrate reasonably well.** Web Forms and WCF don't.
3. **DI is now built-in and mandatory** — this is the #1 paradigm shift.
4. **`HttpContext.Current` is gone** — this causes the most refactoring.
5. **Use the Upgrade Assistant** for mechanical changes, YARP for incremental migration.
6. **Performance gains are real** — 2-10x improvement is well-documented.
7. **Linux hosting saves money** — and opens up containerisation.
8. **Don't migrate for the sake of it** — have a clear business reason.
9. **Plan for the auth rewrite** — same concepts, completely different implementation.
10. **Start with a pilot** — migrate one small service first to build team confidence.

The best migration conversations are honest ones. Know what's easy, know what's hard, and give customers a realistic picture. That's what builds trust.
