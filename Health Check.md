# Health Checks

Here's a detailed explanation and implementation of **Health Checks** in a .NET Core Web API, focusing on the latest versions (e.g., .NET 6, 7, or 8 as of February 27, 2025). Health Checks are essential for monitoring the health of your application and its dependencies, enabling automated systems (like Kubernetes) to determine if the app is functioning correctly.

---

### What are Health Checks?

Health Checks in .NET Core provide a framework to report the status of an application and its dependencies (e.g., databases, external services) through endpoints. They are built into the `Microsoft.Extensions.Diagnostics.HealthChecks` package and support various integrations, such as UI dashboards or orchestration tools.

- **Purpose**: Ensure the app is "healthy" (running correctly) or detect issues like degraded performance or complete failure.
- **Use Cases**: Load balancer routing, Kubernetes pod management, or alerting.

---

### Key Concepts

1. **Health Check Types**:
   - **Healthy**: The system is functioning normally.
   - **Degraded**: The system is operational but with issues (e.g., slow response).
   - **Unhealthy**: The system has failed or cannot perform critical tasks.

2. **Built-in Checks**: Database connectivity, memory usage, etc.
3. **Custom Checks**: Define your own logic for app-specific health.

---

### Implementing Health Checks in .NET Core Web API

#### Step 1: Install Required Packages
The core Health Checks package is included with ASP.NET Core, but you may need additional packages for specific checks:

```bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore  # For EF Core
```

#### Step 2: Configure Health Checks in `Program.cs`
In .NET 6+ (minimal hosting model):

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Add health checks
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy("API is running"), tags: new[] { "api" })
    .AddUrlGroup(new Uri("https://api.example.com"), name: "external-api", tags: new[] { "external" })
    .AddSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"), 
                  name: "sqlserver", 
                  tags: new[] { "database" });

var app = builder.Build();

// Configure the HTTP request pipeline
app.UseRouting();
app.UseAuthorization();

// Map health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true, // Include all checks
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var result = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description
            })
        };
        await context.Response.WriteAsJsonAsync(result);
    }
});

app.MapControllers();

app.Run();
```

- **Explanation**:
  - **`AddHealthChecks()`**: Registers the health check service.
  - **`AddCheck`**: A simple check verifying the API is running.
  - **`AddUrlGroup`**: Checks an external API’s availability.
  - **`AddSqlServer`**: Verifies SQL Server connectivity (requires a connection string in `appsettings.json`).
  - **`MapHealthChecks`**: Exposes the `/health` endpoint with a custom JSON response.

#### Step 3: Add Configuration (e.g., `appsettings.json`)
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb;Trusted_Connection=True;"
  }
}
```

#### Step 4: Test the Health Endpoint
- **Request**: `GET /health`
- **Sample Response**:
  ```json
  {
    "status": "Healthy",
    "checks": [
      {
        "name": "self",
        "status": "Healthy",
        "description": "API is running"
      },
      {
        "name": "external-api",
        "status": "Healthy",
        "description": null
      },
      {
        "name": "sqlserver",
        "status": "Healthy",
        "description": null
      }
    ]
  }
  ```

---

### Advanced Health Check Features

#### 1. **Custom Health Checks**
Create a custom check for application-specific logic:

```csharp
public class MemoryHealthCheck : IHealthCheck
{
    private readonly long _maxMemoryThreshold;

    public MemoryHealthCheck(long maxMemoryThreshold = 1024 * 1024 * 1024) // 1GB default
    {
        _maxMemoryThreshold = maxMemoryThreshold;
    }

    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        var memoryUsed = GC.GetTotalMemory(false);
        if (memoryUsed > _maxMemoryThreshold)
        {
            return Task.FromResult(HealthCheckResult.Degraded($"Memory usage ({memoryUsed / 1024 / 1024} MB) exceeds threshold."));
        }
        return Task.FromResult(HealthCheckResult.Healthy($"Memory usage: {memoryUsed / 1024 / 1024} MB"));
    }
}

// Register in Program.cs
builder.Services.AddHealthChecks()
    .AddCheck<MemoryHealthCheck>("memory", tags: new[] { "system" });
```

- **Explanation**: Checks if memory usage exceeds 1GB, returning `Degraded` if true.

#### 2. **Multiple Endpoints**
Separate endpoints for different purposes (e.g., readiness vs. liveness):

```csharp
// Readiness: Checks if the app is ready to handle requests
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("api") || check.Tags.Contains("database")
});

// Liveness: Checks if the app is alive
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("system")
});
```

- **Use Case**:
  - `/health/ready`: For Kubernetes readiness probes (app can serve traffic).
  - `/health/live`: For liveness probes (app is running).

#### 3. **Health Check UI**
Add a UI dashboard with `AspNetCore.HealthChecks.UI`:

```bash
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.UI.Client
dotnet add package AspNetCore.HealthChecks.UI.InMemory.Storage
```

Update `Program.cs`:

```csharp
builder.Services.AddHealthChecksUI()
    .AddInMemoryStorage();

app.MapHealthChecksUI();
```

- **Access**: `GET /healthchecks-ui` provides a web interface showing health status.

#### 4. **Middleware for Health-Based Routing**
Use health checks to conditionally enable endpoints:

```csharp
app.UseWhen(context => app.Services.GetRequiredService<IHealthCheckService>().CheckHealthAsync().Result.OverallStatus == HealthStatus.Healthy, 
    appBuilder => appBuilder.UseRouting().UseAuthorization().UseEndpoints(endpoints => endpoints.MapControllers()));
```

- **Explanation**: Only enables the API pipeline if the app is healthy.

---

### Best Practices

1. **Tag Checks**: Use tags to group related checks (e.g., "database", "external") for filtering.
2. **Custom Responses**: Tailor the response format for your monitoring tools (e.g., Prometheus).
3. **Timeouts**: Set timeouts on checks (e.g., `AddUrlGroup(..., timeout: TimeSpan.FromSeconds(5))`) to avoid hanging.
4. **Security**: Protect endpoints with authentication/authorization in production:
   ```csharp
   app.MapHealthChecks("/health").RequireAuthorization();
   ```
5. **Monitoring**: Integrate with external systems (e.g., Prometheus, Azure Application Insights) using additional packages.

---

### Real-World Scenario
Imagine an e-commerce API:
- **Self Check**: Verifies the API is running (`/health/live`).
- **Database Check**: Ensures SQL Server is accessible (`/health/ready`).
- **External Payment API**: Monitors third-party service health.
- **Memory Check**: Alerts if memory usage is high (degraded state).

Sample response if the payment API fails:
```json
{
  "status": "Unhealthy",
  "checks": [
    { "name": "self", "status": "Healthy", "description": "API is running" },
    { "name": "sqlserver", "status": "Healthy", "description": null },
    { "name": "external-api", "status": "Unhealthy", "description": "Payment API unavailable" }
  ]
}
```

---

### Integration with Kubernetes
- **Readiness Probe**:
  ```yaml
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 80
    initialDelaySeconds: 5
    periodSeconds: 10
  ```
- **Liveness Probe**:
  ```yaml
  livenessProbe:
    httpGet:
      path: /health/live
      port: 80
    initialDelaySeconds: 15
    periodSeconds: 20
  ```

---

Health Checks in the latest .NET versions are powerful and extensible, making your Web API observable and resilient. Let me know if you’d like to explore a specific check or integration further!
