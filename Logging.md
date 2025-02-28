# Logging

Logging in a .NET Core Web API (covering the latest versions like .NET 6, 7, or 8) is a critical aspect for monitoring, debugging, and auditing application behavior. Microsoft provides a built-in logging framework via the Microsoft.Extensions.Logging package, which is extensible, flexible, and integrates seamlessly with ASP.NET Core. Below is a detailed explanation with examples, focusing on intermediate to advanced usage in the latest .NET versions (as of February 27, 2025).

## Overview of Logging in .NET Core Web API

  - Framework: The logging system is based on the ILogger interface and uses a provider model (e.g., Console, Debug, File, etc.).
  - Dependency Injection: Loggers are injected into classes like controllers or services.
  -  Log Levels: Supports levels like Trace, Debug, Information, Warning, Error, and Critical.
  -  Extensibility: You can add third-party providers like Serilog, NLog, or log4net for advanced features.

## Setting Up Logging

## Step 1: Default Configuration

In .NET 6+ (minimal hosting model), logging is preconfigured in Program.cs:

```
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Configure logging (optional customization)
builder.Logging
    .ClearProviders() // Clears default providers (Console, Debug, etc.)
    .AddConsole()    // Adds console logging
    .AddDebug();     // Adds debug logging

var app = builder.Build();

app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Explanation:
        WebApplication.CreateBuilder automatically adds default logging providers (Console, Debug, EventSource, EventLog on Windows).
        builder.Logging allows customization of providers and settings.

## Step 2: Appsettings Configuration

Control log levels and output via appsettings.json:

```
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",          // Reduce noise from Microsoft logs
      "Microsoft.Hosting.Lifetime": "Information"
    },
    "Console": {
      "LogLevel": {
        "Default": "Debug"
      }
    }
  }
}
```

Explanation:

  - Default: Sets the minimum log level for all categories unless overridden.
  - Microsoft: Filters logs from ASP.NET Core framework components.
  - Provider-specific settings (e.g., Console) can override global settings.

## Using Logging in a Web API
Example: Basic Logging in a Controller

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;

namespace MyWebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class WeatherController : ControllerBase
    {
        private readonly ILogger<WeatherController> _logger;

        // Logger injected via constructor
        public WeatherController(ILogger<WeatherController> logger)
        {
            _logger = logger;
        }

        [HttpGet]
        public IActionResult GetWeather()
        {
            _logger.LogInformation("Fetching weather data at {Time}", DateTime.Now);

            try
            {
                // Simulate fetching weather data
                var weather = new { Temperature = 25, Condition = "Sunny" };
                _logger.LogDebug("Weather data retrieved: {@Weather}", weather);
                return Ok(weather);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error fetching weather data");
                return StatusCode(500, "An error occurred");
            }
        }
    }
}
```

Explanation:

  - Injection: ILogger<WeatherController> is injected, scoped to the WeatherController category.
  - Log Levels: LogInformation for general info, LogDebug for detailed diagnostics, LogError for exceptions.
  - Structured Data: {@Weather} logs the object as structured data (JSON-like), not just a string.
  - Exception Logging: Includes the exception stack trace with LogError(ex, ...).

## Advanced Logging Features

### 1. Structured Logging

Structured logging allows you to log data as key-value pairs, making it easier to query logs in tools like Elasticsearch or Azure Monitor.

```
_logger.LogInformation("User {UserId} accessed resource {ResourceId}", 123, "weather");
```

Output (Console): User 123 accessed resource weather
    With a structured provider (e.g., Serilog), this becomes searchable: { "UserId": 123, "ResourceId": "weather" }.


### 2. Customizing Log Messages

Use EventId for categorizing logs:

```
public static class EventIds
{
    public static readonly EventId DataFetch = new(1001, "DataFetch");
}

_logger.LogInformation(EventIds.DataFetch, "Data fetched successfully");
```

Explanation: EventId helps filter or group logs by specific events.

### 3. Scoped Logging

Log additional context using scopes:

```
using (_logger.BeginScope("RequestId: {RequestId}", Guid.NewGuid()))
{
    _logger.LogInformation("Processing request");
    // All logs within this scope include the RequestId
}
```

Output: [RequestId: <guid>] Processing request

### 4. Filtering Logs

Dynamically filter logs based on category or level:

```
builder.Logging.AddFilter("MyWebApi.Controllers.WeatherController", LogLevel.Debug);
```

Only Debug and higher logs from WeatherController are emitted.
