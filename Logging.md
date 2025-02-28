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

# Implement log4net

What is log4net?

log4net is a popular, open-source logging framework for .NET, originally ported from Apache log4j. It provides a flexible logging system with support for multiple appenders (e.g., console, file, database), log levels, and custom configurations. While .NET Core’s built-in logging (Microsoft.Extensions.Logging) is sufficient for many cases, log4net is favored for its rich configuration options and legacy compatibility.

## Step-by-Step Implementation

### Step 1: Install log4net Packages

Add the necessary NuGet packages to your project:

```
dotnet add package log4net
dotnet add package Microsoft.Extensions.Logging.Log4Net.AspNetCore
```

- log4net: Core logging library.
- Microsoft.Extensions.Logging.Log4Net.AspNetCore: Integrates log4net with the .NET Core logging abstraction (ILogger).

Step 2: Configure log4net

log4net uses an external configuration file (e.g., log4net.config) to define appenders, loggers, and levels.

  - Create log4net.config:
  - Add this file to your project root:

```
<?xml version="1.0" encoding="utf-8"?>
<log4net>
    <!-- Define appenders -->
    <appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
        <layout type="log4net.Layout.PatternLayout">
            <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
        </layout>
    </appender>
    <appender name="FileAppender" type="log4net.Appender.RollingFileAppender">
        <file value="logs/myapp.log" />
        <appendToFile value="true" />
        <rollingStyle value="Size" />
        <maxSizeRollBackups value="5" />
        <maximumFileSize value="10MB" />
        <layout type="log4net.Layout.PatternLayout">
            <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
        </layout>
    </appender>

    <!-- Root logger configuration -->
    <root>
        <level value="DEBUG" />
        <appender-ref ref="ConsoleAppender" />
        <appender-ref ref="FileAppender" />
    </root>

    <!-- Optional: Specific logger for a namespace -->
    <logger name="MyWebApi.Controllers">
        <level value="INFO" />
    </logger>
</log4net>
```

Explanation:
  - ConsoleAppender: Logs to the console.
  - FileAppender: Logs to a rolling file (logs/myapp.log), with a max size of 10MB and up to 5 backups.
  - Root: Default logger with DEBUG level, using both appenders.
  - Specific Logger: Overrides the root for MyWebApi.Controllers to log only INFO and above.
  - Pattern: Formats logs as date [thread] level logger - message.

Set File Properties:
        In Visual Studio, right-click log4net.config > Properties > Set "Copy to Output Directory" to "Copy if newer" or "Copy always".

### Step 3: Integrate log4net with .NET Core

Modify Program.cs to configure log4net as a logging provider:

```
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Configure logging with log4net
builder.Logging.ClearProviders(); // Optional: Clear default providers
builder.Logging.AddLog4Net("log4net.config"); // Load log4net configuration

var app = builder.Build();

// Configure the HTTP request pipeline
app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Explanation:
- AddLog4Net: Registers log4net as a provider, pointing to log4net.config.
- ClearProviders: Removes default providers (Console, Debug) if you want log4net exclusively.

### Step 4: Use Logging in a Controller

Inject and use ILogger as you would with the built-in framework:

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

        public WeatherController(ILogger<WeatherController> logger)
        {
            _logger = logger;
        }

        [HttpGet]
        public IActionResult GetWeather()
        {
            _logger.LogDebug("Starting weather fetch at {Time}", DateTime.Now);
            _logger.LogInformation("Fetching weather data");

            try
            {
                var weather = new { Temperature = 25, Condition = "Sunny" };
                _logger.LogInformation("Weather fetched successfully: {@Weather}", weather);
                return Ok(weather);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to fetch weather data");
                return StatusCode(500, "An error occurred");
            }
        }
    }
}
```

Output (assuming DEBUG level):

Console:

```
2025-02-27 10:00:00 [1] DEBUG MyWebApi.Controllers.WeatherController - Starting weather fetch at 2025-02-27 10:00:00
2025-02-27 10:00:00 [1] INFO  MyWebApi.Controllers.WeatherController - Fetching weather data
2025-02-27 10:00:00 [1] INFO  MyWebApi.Controllers.WeatherController - Weather fetched successfully: {"Temperature":25,"Condition":"Sunny"}
```

File (logs/myapp.log): Same content appended.

# Advanced Configuration and Usage

### 1. Dynamic Configuration

Load log4net config programmatically instead of a file:

```
using log4net;
using log4net.Config;

var logRepository = LogManager.GetRepository(Assembly.GetEntryAssembly());
XmlConfigurator.Configure(logRepository, new FileInfo("log4net.config"));
builder.Logging.AddLog4Net(); // No file path needed here
```

### 2. Custom Appenders

Add a database appender (e.g., SQL Server):

```
<appender name="AdoNetAppender" type="log4net.Appender.AdoNetAppender">
    <bufferSize value="1" />
    <connectionType value="System.Data.SqlClient.SqlConnection, System.Data" />
    <connectionString value="Server=localhost;Database=Logs;Trusted_Connection=True;" />
    <commandText value="INSERT INTO Logs ([Date],[Thread],[Level],[Logger],[Message],[Exception]) VALUES (@log_date, @thread, @log_level, @logger, @message, @exception)" />
    <parameter>
        <parameterName value="@log_date" />
        <dbType value="DateTime" />
        <layout type="log4net.Layout.RawTimeStampLayout" />
    </parameter>
    <!-- Add other parameters similarly -->
</appender>
<root>
    <appender-ref ref="AdoNetAppender" />
</root>
```

### 3. Scoped Logging

Use LogContext for additional context:

```
using (log4net.LogicalThreadContext.Properties["RequestId"] = Guid.NewGuid().ToString())
{
    _logger.LogInformation("Processing request");
}
```

Output: Adds RequestId to the log pattern (update <conversionPattern> with %property{RequestId}).

### 4. Filter Logs

Filter out specific levels or messages:

```
<appender name="ConsoleAppender" type="log4net.Appender.ConsoleAppender">
    <filter type="log4net.Filter.LevelRangeFilter">
        <levelMin value="INFO" />
        <levelMax value="ERROR" />
    </filter>
    <layout type="log4net.Layout.PatternLayout">
        <conversionPattern value="%date %-5level - %message%newline" />
    </layout>
</appender>
```

Only logs INFO to ERROR levels.

### Troubleshooting

  No Logs?: Ensure log4net.config is copied to the output directory and the path in AddLog4Net is correct.
  Debugging: Enable log4net internal debugging:

```
<log4net debug="true" />
```




