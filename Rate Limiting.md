# Rate Limiting

Rate Limiting in a .NET Core Web API (covering the latest versions like .NET 6, 7, or 8 as of February 27, 2025) is a critical feature for protecting your API from abuse, ensuring fair usage, and maintaining performance under load. Starting with .NET 7, Microsoft introduced built-in rate limiting support via the Microsoft.AspNetCore.RateLimiting package, making it easier to implement without relying solely on third-party libraries. Below, I’ll explain rate limiting in detail, including its implementation, options, and examples.

# What is Rate Limiting?

Rate Limiting restricts the number of requests a client (e.g., by IP, user, or token) can make to an API within a specified time window. It helps:

  - Prevent Denial-of-Service (DoS) attacks.
  - Manage resource usage and costs.
  - Enforce API usage quotas (e.g., for free-tier users).

## Rate Limiting in .NET Core
## Built-in Support (.NET 7+)

The Microsoft.AspNetCore.RateLimiting package provides middleware and policies for rate limiting, with four main algorithms:

  - Fixed Window: Limits requests in a fixed time window (e.g., 100 requests per minute).
  - Sliding Window: Similar to fixed window but slides the window for smoother limits.
  - Token Bucket: Allows bursts up to a capacity, refilling tokens over time.
  - Concurrency: Limits the number of concurrent requests.

## Third-Party Alternatives

For .NET versions before 7 or for advanced scenarios, libraries like AspNetCoreRateLimit are still popular.

## Implementing Rate Limiting in .NET 8 Web API
### Step 1: Install the Package

The rate limiting package is included with ASP.NET Core in .NET 7+, so no additional installation is needed for the latest versions. Verify your project targets .NET 8:

```
<TargetFramework>net8.0</TargetFramework>
```

Step 2: Configure Rate Limiting in Program.cs

```
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Configure rate limiting
builder.Services.AddRateLimiter(options =>
{
    // Fixed Window Policy
    options.AddFixedWindowLimiter("fixed", new FixedWindowRateLimiterOptions
    {
        PermitLimit = 10,               // 10 requests allowed
        Window = TimeSpan.FromMinutes(1), // Per minute
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        QueueLimit = 5                  // Queue up to 5 requests if limit exceeded
    });

    // Token Bucket Policy
    options.AddTokenBucketLimiter("token", new TokenBucketRateLimiterOptions
    {
        TokenLimit = 20,                // Max burst capacity
        QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
        QueueLimit = 2,
        ReplenishmentPeriod = TimeSpan.FromSeconds(10), // Refill every 10 seconds
        TokensPerPeriod = 5             // Refill 5 tokens per period
    });

    // Global rejection response
    options.RejectionStatusCode = 429; // Too Many Requests
    options.OnRejected = async (context, cancellationToken) =>
    {
        context.HttpContext.Response.Headers["Retry-After"] = "60"; // Retry after 60 seconds
        await context.HttpContext.Response.WriteAsync("Too many requests. Please try again later.", cancellationToken);
    };
});

var app = builder.Build();

// Apply rate limiting globally
app.UseRateLimiter();

// Configure the HTTP request pipeline
app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Explanation:

  - AddRateLimiter: Registers the rate limiting service.
  - FixedWindowLimiter: Limits to 10 requests per minute, queuing up to 5 excess requests.
  - TokenBucketLimiter: Allows bursts up to 20 requests, refilling 5 tokens every 10 seconds.
  - OnRejected: Customizes the rejection response with a 429 status and a Retry-After header.
  - UseRateLimiter: Applies rate limiting middleware to all requests.

## Step 3: Apply Rate Limiting to Endpoints

Use attributes or policies to apply rate limiting selectively:

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.RateLimiting;

namespace MyWebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class WeatherController : ControllerBase
    {
        [HttpGet]
        [EnableRateLimiting("fixed")] // Apply fixed window policy
        public IActionResult GetWeather()
        {
            return Ok(new { Temperature = 25, Condition = "Sunny" });
        }

        [HttpPost]
        [EnableRateLimiting("token")] // Apply token bucket policy
        public IActionResult PostWeather([FromBody] object data)
        {
            return Ok("Weather data posted");
        }

        [HttpGet("unlimited")]
        [DisableRateLimiting] // Bypass rate limiting
        public IActionResult GetUnlimited()
        {
            return Ok("No rate limit applied");
        }
    }
}
```

### Explanation:

  - [EnableRateLimiting]: Applies a named policy (e.g., "fixed" or "token") to specific actions.
  - [DisableRateLimiting]: Excludes an endpoint from global rate limiting.

## Step 4: Test the API

  - Request: GET /weather (10 times in a minute)
    - First 10 succeed with 200 OK.
    - 11th request returns 429 Too Many Requests with:

```
Too many requests. Please try again later.
Retry-After: 60
```

- Request: POST /weather (burst of 21 requests)
  - First 20 succeed (token bucket capacity), then 429 until tokens replenish.

# Advanced Rate Limiting Features

## 1. Partitioned Rate Limiting

Limit based on client-specific keys (e.g., IP address, user ID):

```
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("ip-based", new FixedWindowRateLimiterOptions
    {
        PermitLimit = 5,
        Window = TimeSpan.FromMinutes(1),
        QueueLimit = 0,
        AutoReplenishment = true
    }).PartitionedByHttpContext(context => context.Connection.RemoteIpAddress?.ToString() ?? "unknown");
});
```

Explanation: Each IP gets its own 5-request limit per minute.

## 2. Sliding Window

For smoother throttling:

```
options.AddSlidingWindowLimiter("sliding", new SlidingWindowRateLimiterOptions
{
    PermitLimit = 10,
    Window = TimeSpan.FromMinutes(1),
    SegmentsPerWindow = 2, // Divide window into two 30-second segments
    QueueLimit = 2
});
```

Explanation: Tracks requests across 30-second segments, allowing a more even distribution.

## 3. Dynamic Limits

Adjust limits based on runtime conditions:

```
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("dynamic", partition =>
    {
        var userId = partition.HttpContext.User.Identity?.Name ?? "anonymous";
        return userId == "admin" 
            ? RateLimitPartition.GetFixedWindowLimiter(userId, _ => new FixedWindowRateLimiterOptions { PermitLimit = 100, Window = TimeSpan.FromMinutes(1) })
            : RateLimitPartition.GetFixedWindowLimiter(userId, _ => new FixedWindowRateLimiterOptions { PermitLimit = 10, Window = TimeSpan.FromMinutes(1) });
    });
});
```

Explanation: Admins get 100 requests/min, others get 10.

## 4. Custom Response Headers

Add detailed rate limit info:

```
options.OnRejected = async (context, token) =>
{
    context.HttpContext.Response.StatusCode = 429;
    context.HttpContext.Response.Headers["X-Rate-Limit"] = "10";
    context.HttpContext.Response.Headers["X-Rate-Limit-Remaining"] = "0";
    context.HttpContext.Response.Headers["Retry-After"] = "60";
    await context.HttpContext.Response.WriteAsync("Rate limit exceeded.", token);
};
```

# Real-World Scenario

### Imagine an e-commerce API:

 - Public Endpoint (GET /products): Fixed window, 50 requests/minute per IP.
 - Authenticated Endpoint (POST /orders): Token bucket, 20 burst capacity, 5 tokens/10 seconds per user.
 - Admin Endpoint (GET /reports): Dynamic policy, 500 requests/minute for admins.

## Sample rejection:

```
HTTP/1.1 429 Too Many Requests
X-Rate-Limit: 50
X-Rate-Limit-Remaining: 0
Retry-After: 60
Content: "Rate limit exceeded."
```
