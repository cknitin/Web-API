# Exception Handling

In .NET Core Web API (and general software development), error handling can be implemented in various ways depending on the granularity, scope, and purpose of the error management. Below are the types of error handling commonly used in a Web API context, with explanations and examples tailored to .NET Core. I'll also tie this back to your previous request for examples.

## 1. Manual Error Handling (Try-Catch Blocks)
Description: Explicitly handling exceptions within specific code blocks using try-catch. This gives developers full control over how errors are caught and responded to.
When to Use: For specific business logic errors or when you expect certain exceptions in a particular method or action.
Pros: Fine-grained control, ideal for localized error handling.
Cons: Can lead to repetitive code if overused across the application.
Example:

```
[HttpGet("{id}")]
public IActionResult GetWeather(int id)
{
    try
    {
        if (id <= 0)
            return BadRequest("ID must be positive.");
        
        var data = FetchWeatherData(id); // Might throw an exception
        return Ok(data);
    }
    catch (InvalidOperationException ex)
    {
        return StatusCode(503, $"Service unavailable: {ex.Message}");
    }
    catch (Exception ex)
    {
        return StatusCode(500, $"Unexpected error: {ex.Message}");
    }
}

private string FetchWeatherData(int id)
{
    if (id == 42) throw new InvalidOperationException("Weather service down.");
    return $"Weather for {id}";
}
```

# 2. Global Exception Handling (Middleware)

Description: A centralized mechanism to catch all unhandled exceptions in the application pipeline using middleware. This ensures a consistent error response format.
When to Use: As a safety net for unhandled exceptions across the entire API.
Pros: Reduces boilerplate code, enforces uniformity.
Cons: Less control over specific exceptions unless combined with other methods.
Example:

```
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async context =>
        {
            var errorFeature = context.Features.Get<IExceptionHandlerFeature>();
            var exception = errorFeature?.Error;

            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/json";

            var errorResponse = new
            {
                StatusCode = 500,
                Message = "An error occurred.",
                Details = env.IsDevelopment() ? exception?.ToString() : null
            };

            await context.Response.WriteAsync(JsonSerializer.Serialize(errorResponse));
        });
    });

    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

## 3. Exception Filters

Description: Custom filters that intercept exceptions at the controller or action level before they reach the global handler. They’re part of the ASP.NET Core MVC pipeline.
When to Use: For handling specific exception types in a reusable way across controllers or actions.
Pros: Reusable, scoped to controllers or actions, integrates with MVC pipeline.
Cons: Requires defining filter classes, less flexible than middleware for global scenarios.
Example:

```
public class CustomExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        if (context.Exception is ArgumentException ex)
        {
            context.Result = new ObjectResult(new { Error = ex.Message })
            {
                StatusCode = 400
            };
            context.ExceptionHandled = true;
        }
    }
}

[ApiController]
[Route("[controller]")]
[TypeFilter(typeof(CustomExceptionFilter))] // Apply filter
public class WeatherController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        if (id < 0) throw new ArgumentException("ID cannot be negative.");
        return Ok($"Weather for {id}");
    }
}
```

## 4. Model Validation Error Handling

Description: Automatically handles validation errors for incoming data using the [ApiController] attribute and model validation attributes.
When to Use: For validating request data (e.g., DTOs or models) before processing.
Pros: Built-in, reduces manual validation code.
Cons: Limited to model binding errors, not general exceptions.
Example:

```
public class WeatherRequest
{
    [Required]
    [Range(1, int.MaxValue, ErrorMessage = "ID must be positive.")]
    public int Id { get; set; }
}

[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpPost]
    public IActionResult Post(WeatherRequest request)
    {
        if (!ModelState.IsValid)
        {
            return BadRequest(ModelState); // Automatically handled by [ApiController]
        }
        return Ok($"Weather for {request.Id}");
    }
}
```

```
Output (if Id is missing or invalid):
{
  "Id": ["The Id field is required."]
}
```


## 5. Action Result-Based Error Handling
Description: Returning specific IActionResult types (e.g., BadRequest(), NotFound()) to handle errors without throwing exceptions.
When to Use: For expected error conditions that don’t warrant exceptions (e.g., resource not found).
Pros: Clean, aligns with HTTP semantics, no exception overhead.
Cons: Not suitable for unexpected failures.
Example:

```
[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    private readonly Dictionary<int, string> _weatherData = new() { { 1, "Sunny" } };

    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        if (!_weatherData.ContainsKey(id))
            return NotFound($"Weather data for ID {id} not found.");
        
        return Ok(_weatherData[id]);
    }
}
```

## 6. Custom Exception Types
Description: Defining and throwing custom exceptions to represent specific error conditions, then handling them elsewhere (e.g., filters or middleware).
When to Use: For domain-specific errors that need distinct handling or logging.
Pros: Provides semantic clarity, reusable across the app.
Cons: Increases complexity with additional exception classes.
Example:

```
public class WeatherServiceException : Exception
{
    public WeatherServiceException(string message) : base(message) { }
}

[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        if (id == 42)
            throw new WeatherServiceException("Weather service is temporarily unavailable.");
        
        return Ok($"Weather for {id}");
    }
}

// Middleware to handle it
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var error = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        if (error is WeatherServiceException wse)
        {
            context.Response.StatusCode = 503;
            await context.Response.WriteAsync(JsonSerializer.Serialize(new { Message = wse.Message }));
        }
        else
        {
            context.Response.StatusCode = 500;
            await context.Response.WriteAsync(JsonSerializer.Serialize(new { Message = "Unknown error" }));
        }
    });
});

```

## 7. Problem Details (RFC 7807)
Description: A standardized way to return error details using the ProblemDetails class, often integrated with middleware or filters.
When to Use: For RESTful APIs where consistent, machine-readable error responses are required.
Pros: Follows a standard, interoperable with clients.
Cons: Slightly more verbose than simple JSON responses.
Example:

```
public void Configure(IApplicationBuilder app)
{
    app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async context =>
        {
            var error = context.Features.Get<IExceptionHandlerFeature>()?.Error;
            var problemDetails = new ProblemDetails
            {
                Status = 500,
                Title = "Server Error",
                Detail = error?.Message,
                Instance = context.Request.Path
            };
            context.Response.StatusCode = 500;
            context.Response.ContentType = "application/problem+json";
            await context.Response.WriteAsync(JsonSerializer.Serialize(problemDetails));
        });
    });

    app.UseRouting();
    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

```
Output:
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "Server Error",
  "status": 500,
  "detail": "Something went wrong.",
  "instance": "/weather/42"
}
```

## Comparison of Error Handling Types

| Type                  | Scope             | Granularity       | Complexity | Use Case Example                   |
|-----------------------|-------------------|-------------------|------------|------------------------------------|
| Manual (Try-Catch)    | Action/Method     | High             | Low        | Specific business logic errors     |
| Global Middleware     | Application       | Low              | Medium     | Unhandled exceptions               |
| Exception Filters     | Controller/Action | Medium           | Medium     | Reusable exception handling        |
| Model Validation      | Action            | Medium           | Low        | Input validation errors            |
| Action Results        | Action            | High             | Low        | Expected conditions (e.g., 404)    |
| Custom Exceptions     | Application       | Medium           | High       | Domain-specific error scenarios    |
| Problem Details       | Application       | Medium           | Medium     | Standardized error responses       |

Best Practices
  - Layered Approach: Combine global middleware (for unexpected errors) with manual handling or filters (for expected errors).
  - Environment-Specific: Show detailed errors (e.g., stack traces) in Development, but mask them in Production.
  - Logging: Always log exceptions (e.g., using ILogger) for debugging and monitoring.
  - Consistency: Use a unified error response format (e.g., ErrorDetails or ProblemDetails) across all handlers.
  - This covers the main types of error handling in .NET Core Web API. Each has its strengths, and the choice depends on your application’s needs—whether it’s simplicity, scalability, or standardization. Let me know if you’d like a deeper dive into any specific type!
