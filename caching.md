## Caching in webapi .net core

Caching in .NET Core Web API can significantly improve the performance of your application by reducing database load, decreasing response times, and minimizing network traffic. Here's an overview of how caching can be implemented:
Types of Caching in .NET Core

### 1. In-Memory Caching:
        Stores data in the server's memory, which is fast but limited to the server's memory capacity and lost when the application restarts.
### 2. Distributed Caching:
        Uses an external cache like Redis or SQL Server to store data across multiple servers, enhancing scalability and persistence.
### 3. Response Caching:
        Caches HTTP responses at the server or client level to speed up subsequent requests.    

## 1. Implementing In-Memory Caching

Step 1: Add the Memory Cache Service
Add IMemoryCache to your Startup.cs in the ConfigureServices method:

```
public void ConfigureServices(IServiceCollection services)
{
    services.AddMemoryCache();
    services.AddControllers();
}
```

Step 2: Use the Cache in Your Controller

```
using Microsoft.Extensions.Caching.Memory;

[ApiController]
[Route("[controller]")]
public class ValuesController : ControllerBase
{
    private readonly IMemoryCache _cache;

    public ValuesController(IMemoryCache memoryCache)
    {
        _cache = memoryCache;
    }

    [HttpGet]
    public IActionResult Get()
    {
        if (!_cache.TryGetValue("someKey", out string cachedValue))
        {
            // Key not in cache, so get data from your source and cache it
            cachedValue = GetDataFromSource();

            var cacheEntryOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromSeconds(30)); // Cache for 30 seconds

            _cache.Set("someKey", cachedValue, cacheEntryOptions);
        }

        return Ok(cachedValue);
    }

    private string GetDataFromSource()
    {
        // Simulate fetching data from a database or external service
        return "Data from source";
    }
}
```

## 2. Distributed Caching with Redis

Step 1: Install Necessary Packages

```
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

### Step 2: Configure in Startup.cs

```
public void ConfigureServices(IServiceCollection services)
{
    services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = "localhost"; // Connection string to Redis
        options.InstanceName = "SampleInstance"; // Optional
    });
    services.AddControllers();
}
```

### Step 3: Use Distributed Cache

```
using Microsoft.Extensions.Caching.Distributed;

[ApiController]
[Route("[controller]")]
public class ValuesController : ControllerBase
{
    private readonly IDistributedCache _cache;

    public ValuesController(IDistributedCache cache)
    {
        _cache = cache;
    }

    [HttpGet]
    public async Task<IActionResult> Get()
    {
        string cachedValue = await _cache.GetStringAsync("someKey");

        if (cachedValue == null)
        {
            cachedValue = GetDataFromSource();

            await _cache.SetStringAsync("someKey", cachedValue, new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromSeconds(30)
            });
        }

        return Ok(cachedValue);
    }

    private string GetDataFromSource()
    {
        return "Data from source";
    }
}
```

## 3. Response Caching

    Enable Response Caching in Startup.cs:

```
public void ConfigureServices(IServiceCollection services)
{
    services.AddResponseCaching();
    services.AddControllers();
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseResponseCaching();
    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

### Use [ResponseCache] Attribute:

```
[ApiController]
[Route("[controller]")]
public class ValuesController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 30)]
    public IActionResult Get()
    {
        return Ok("This response will be cached for 30 seconds");
    }
}
```

## Considerations:

  - Cache Invalidation: Think about how to update or invalidate cache entries when underlying data changes.
  - Cache Key Management: Use unique keys for different cache entries to avoid conflicts.
  - Security: Ensure sensitive data isn't cached or cached data remains secure.
  - Scaling: In-memory caching might not scale well across multiple instances; consider distributed caching for better scalability.

Implementing caching should be done judiciously, balancing the performance benefits with the complexity of maintaining cache consistency.
