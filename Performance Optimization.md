# Performance optimization

Here's a detailed explanation and implementation of **Performance Optimization** techniques for a .NET Core Web API, focusing on the latest versions (e.g., .NET 6, 7, or 8 as of February 27, 2025). Optimizing performance is crucial for ensuring your API responds quickly, scales efficiently, and uses resources effectively.

---

### Why Optimize Performance?

Performance optimization in a Web API aims to:
- Reduce response times for better user experience.
- Increase throughput to handle more requests.
- Minimize resource usage (CPU, memory, network) for cost efficiency.
- Ensure scalability for high-traffic scenarios.

---

### Key Areas of Optimization

1. **HTTP and Middleware**
2. **Serialization**
3. **Data Access**
4. **Caching**
5. **Asynchronous Programming**
6. **Memory Management**
7. **Response Compression**

Below, I'll explain each with examples tailored to .NET 8.

---

### 1. HTTP and Middleware Optimization

#### Minimize Middleware
Only include necessary middleware to reduce the request pipeline overhead:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

// Minimal pipeline
app.UseRouting();
app.UseAuthorization(); // Only if needed
app.MapControllers();

app.Run();
```

- **Explanation**: Avoid unused middleware (e.g., `UseStaticFiles` if not serving files).

#### Use HTTP/2 or HTTP/3
Enable HTTP/2 (default in .NET Core) or HTTP/3 for multiplexing and reduced latency:

```csharp
builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000, listenOptions =>
    {
        listenOptions.Protocols = Microsoft.AspNetCore.Server.Kestrel.Core.HttpProtocols.Http2;
    });
});
```

- **Explanation**: HTTP/2 reduces head-of-line blocking; HTTP/3 (QUIC) further improves performance over unreliable networks.

---

### 2. Serialization Optimization

#### Use System.Text.Json Efficiently
Optimize JSON serialization (default in .NET Core):

```csharp
builder.Services.AddControllers()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.PropertyNamingPolicy = null; // Avoid camelCase conversion if not needed
        options.JsonSerializerOptions.WriteIndented = false; // Disable pretty-printing
        options.JsonSerializerOptions.DefaultIgnoreCondition = System.Text.Json.Serialization.JsonIgnoreCondition.WhenWritingNull; // Skip nulls
    });
```

- **Controller Example**:
```csharp
[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpGet]
    public IActionResult GetWeather()
    {
        return Ok(new { Temperature = 25, Condition = "Sunny", NullableField = (string?)null });
    }
}
```

- **Response**: `{"Temperature":25,"Condition":"Sunny"}` (no null fields, compact).

#### Source Generators
Use source-generated serializers for better performance (introduced in .NET 6):

```csharp
[JsonSerializable(typeof(WeatherData))]
public partial class WeatherContext : JsonSerializerContext { }

public class WeatherData
{
    public int Temperature { get; set; }
    public string Condition { get; set; }
}

[HttpGet]
public IActionResult GetWeather()
{
    var weather = new WeatherData { Temperature = 25, Condition = "Sunny" };
    return new JsonResult(weather, WeatherContext.Default.WeatherData);
}
```

- **Explanation**: Source generators avoid reflection, improving serialization speed.

---

### 3. Data Access Optimization

#### Use EF Core Efficiently
Optimize Entity Framework Core queries:

```csharp
using Microsoft.EntityFrameworkCore;

public class WeatherDbContext : DbContext
{
    public DbSet<WeatherData> Weather { get; set; }
    // Constructor and configuration omitted for brevity
}

[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    private readonly WeatherDbContext _dbContext;

    public WeatherController(WeatherDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    [HttpGet]
    public async Task<IActionResult> GetWeather()
    {
        var weather = await _dbContext.Weather
            .AsNoTracking() // Disable change tracking for read-only
            .Select(w => new { w.Temperature, w.Condition }) // Project only needed fields
            .Take(10) // Limit results
            .ToListAsync();
        return Ok(weather);
    }
}
```

- **Explanation**:
  - `AsNoTracking`: Reduces memory overhead for read-only queries.
  - `Select`: Avoids fetching unnecessary columns.
  - `Take`: Limits data transfer.

#### Connection Pooling
Reuse database connections (enabled by default in EF Core) and avoid opening/closing connections manually.

---

### 4. Caching

#### Response Caching
Cache HTTP responses:

```csharp
builder.Services.AddResponseCaching();

app.UseResponseCaching();

[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 60, VaryByQueryKeys = new[] { "location" })]
    public IActionResult GetWeather(string location)
    {
        return Ok(new { Temperature = 25, Condition = "Sunny", Location = location });
    }
}
```

- **Explanation**: Caches responses for 60 seconds, varying by `location` query parameter.

#### In-Memory Caching
Cache frequently accessed data:

```csharp
builder.Services.AddMemoryCache();

public class WeatherController : ControllerBase
{
    private readonly IMemoryCache _cache;

    public WeatherController(IMemoryCache cache)
    {
        _cache = cache;
    }

    [HttpGet]
    public IActionResult GetWeather()
    {
        if (!_cache.TryGetValue("weather", out var weather))
        {
            weather = new { Temperature = 25, Condition = "Sunny" };
            _cache.Set("weather", weather, TimeSpan.FromMinutes(5));
        }
        return Ok(weather);
    }
}
```

- **Explanation**: Reduces redundant computation or DB calls.

---

### 5. Asynchronous Programming

#### Use Async/Await
Ensure all I/O operations are asynchronous:

```csharp
[HttpGet]
public async Task<IActionResult> GetWeatherAsync()
{
    var weather = await FetchWeatherAsync(); // Simulate async call
    return Ok(weather);
}

private async Task<object> FetchWeatherAsync()
{
    await Task.Delay(100); // Simulate async I/O
    return new { Temperature = 25, Condition = "Sunny" };
}
```

- **Explanation**: Frees up threads during I/O, improving scalability.

#### ConfigureAwait(false)
Optimize library calls:

```csharp
private async Task<object> FetchWeatherAsync()
{
    await Task.Delay(100).ConfigureAwait(false);
    return new { Temperature = 25, Condition = "Sunny" };
}
```

- **Explanation**: Avoids context switching in non-UI scenarios.

---

### 6. Memory Management

#### Avoid Large Object Allocations
Use `ValueTask` for lightweight async operations:

```csharp
[HttpGet]
public ValueTask<IActionResult> GetWeather()
{
    return new ValueTask<IActionResult>(Ok(new { Temperature = 25, Condition = "Sunny" }));
}
```

- **Explanation**: Reduces heap allocations for simple responses.

#### String Optimization
Use `StringBuilder` or pooled buffers for large strings:

```csharp
using System.Text;

[HttpGet("report")]
public IActionResult GetReport()
{
    var sb = new StringBuilder();
    for (int i = 0; i < 1000; i++)
    {
        sb.Append($"Weather {i}, ");
    }
    return Ok(sb.ToString());
}
```

- **Explanation**: Avoids multiple string allocations.

---

### 7. Response Compression

#### Enable Compression
Reduce payload size:

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>(); // Brotli is more efficient than Gzip
});

app.UseResponseCompression();
```

- **Test**: 
  - Request with `Accept-Encoding: br` → Compressed response.
  - Reduces bandwidth usage significantly for large JSON payloads.

---

### Best Practices

1. **Profile First**: Use tools like BenchmarkDotNet or Visual Studio Profiler to identify bottlenecks.
2. **Minimize Payloads**: Return only necessary data (e.g., use DTOs).
3. **Scale Horizontally**: Optimize for stateless operation to leverage load balancers.
4. **Monitor Performance**: Integrate with Application Insights or Prometheus:
   ```csharp
   builder.Services.AddApplicationInsightsTelemetry();
   ```
5. **Avoid Blocking**: Never use `.Result` or `.Wait()` on async calls.

---

### Real-World Scenario
Imagine an e-commerce API:
- **Product List**: Cached with response caching, compressed, and fetched asynchronously from the DB.
- **Order Submission**: Uses minimal JSON serialization and async I/O.
- **Reports**: Optimized with `StringBuilder` and memory-efficient queries.

Sample optimized endpoint:
```csharp
[HttpGet("products")]
[ResponseCache(Duration = 30)]
public async Task<IActionResult> GetProductsAsync([FromServices] WeatherDbContext db, [FromServices] IMemoryCache cache)
{
    if (!cache.TryGetValue("products", out var products))
    {
        products = await db.Weather.AsNoTracking().Select(p => new { p.Temperature }).ToListAsync();
        cache.Set("products", products, TimeSpan.FromSeconds(30));
    }
    return Ok(products);
}
```

---

Performance optimization in .NET 8 Web API combines built-in features (e.g., compression, async) with best practices (e.g., caching, minimal allocations). Let me know if you’d like to dive deeper into a specific technique or tool!
