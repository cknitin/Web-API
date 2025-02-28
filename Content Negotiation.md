# What is Content Negotiation?

Content Negotiation is the process by which a client and server agree on the format and structure of data exchanged in an HTTP response. In ASP.NET Core, this is handled automatically based on HTTP headers like Accept and configuration settings, ensuring flexibility for diverse clients (e.g., browsers, mobile apps, or other APIs).

- Key Headers:
    - Accept: Specifies the media types the client prefers (e.g., application/json, application/xml).
    - Content-Type: Indicates the media type of the request body (for POST/PUT).
    - Purpose: Allows the API to serve multiple formats without changing endpoints.

#Content Negotiation in .NET Core Web API

## Default Behavior

ASP.NET Core Web APIs default to JSON (application/json) via the System.Text.Json serializer. However, you can extend or customize this behavior to support additional formats like XML, YAML, or custom media types.
Implementing Content Negotiation in .NET 8 Web API

### Step 1: Basic Setup

In .NET 6+ (minimal hosting model), JSON is enabled by default:

```
var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

var app = builder.Build();

app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

Default: AddControllers() includes JSON output via System.Text.Json.

### Step 2: Enable XML Support

To support XML alongside JSON:

```
builder.Services.AddControllers()
    .AddXmlSerializerFormatters(); // Adds XML support
```

Test:

  - Request: GET /weather with Accept: application/json
       - Response: {"temperature": 25, "condition": "Sunny"}
  - Request: GET /weather with Accept: application/xml
       - Response:

```
<WeatherData>
  <Temperature>25</Temperature>
  <Condition>Sunny</Condition>
</WeatherData>
```

### Step 3: Controller Example

```
using Microsoft.AspNetCore.Mvc;

namespace MyWebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class WeatherController : ControllerBase
    {
        [HttpGet]
        public IActionResult GetWeather()
        {
            var weather = new WeatherData { Temperature = 25, Condition = "Sunny" };
            return Ok(weather); // Format determined by Accept header
        }
    }

    public class WeatherData
    {
        public int Temperature { get; set; }
        public string Condition { get; set; }
    }
}
```

Explanation: The response format depends on the Accept header. If unspecified, it defaults to JSON.

# Advanced Content Negotiation

### 1. Custom Media Types

Support a custom format (e.g., application/vnd.myapp.weather+json):

```
builder.Services.AddControllers(options =>
{
    options.ReturnHttpNotAcceptable = true; // Return 406 if format unsupported
    options.RespectBrowserAcceptHeader = true; // Honor browser preferences
})
.AddXmlSerializerFormatters()
.AddJsonOptions(options =>
{
    options.JsonSerializerOptions.PropertyNamingPolicy = null; // Preserve casing
});
```

- Test:

  - Accept: application/vnd.myapp.weather+json → JSON response.
  - Accept: application/xml → XML response.
  - Accept: text/plain → 406 Not Acceptable (unless added).

### 2. Custom Output Formatter

Create a custom formatter for a format like YAML:

- 1. Install a YAML library:

```
dotnet add package YamlDotNet
```
- 2. Define the formatter:

```
using Microsoft.AspNetCore.Mvc.Formatters;
using Microsoft.Net.Http.Headers;
using System.Text;
using YamlDotNet.Serialization;

public class YamlOutputFormatter : TextOutputFormatter
{
    private readonly ISerializer _serializer;

    public YamlOutputFormatter()
    {
        _serializer = new SerializerBuilder().Build();
        SupportedMediaTypes.Add(MediaTypeHeaderValue.Parse("application/x-yaml"));
        SupportedEncodings.Add(Encoding.UTF8);
    }

    protected override bool CanWriteType(Type? type) => true;

    public override async Task WriteResponseBodyAsync(OutputFormatterWriteContext context, Encoding selectedEncoding)
    {
        var response = context.HttpContext.Response;
        var yaml = _serializer.Serialize(context.Object);
        await response.WriteAsync(yaml, selectedEncoding);
    }
}
```

- 3. Register the formatter

```
builder.Services.AddControllers(options =>
{
    options.OutputFormatters.Add(new YamlOutputFormatter());
});
```

Test:

    Accept: application/x-yaml
        Response:

```
Temperature: 25
Condition: Sunny
```

### 3. Query String Negotiation

Allow format selection via a query parameter (e.g., ?format=xml):

```
builder.Services.AddControllers(options =>
{
    options.FormatterMappings.SetMediaTypeMappingForFormat("xml", "application/xml");
    options.FormatterMappings.SetMediaTypeMappingForFormat("json", "application/json");
    options.FormatterMappings.SetMediaTypeMappingForFormat("yaml", "application/x-yaml");
})
.AddXmlSerializerFormatters()
.AddApplicationPart(typeof(YamlOutputFormatter).Assembly); // For custom formatter
```

Test:
        GET /weather?format=xml → XML response.
        GET /weather?format=yaml → YAML response.

### 4. Versioning via Media Types

Combine content negotiation with API versioning:

```
builder.Services.AddApiVersioning(options =>
{
    options.ReportApiVersions = true;
})
.AddMvc(options =>
{
    options.OutputFormatters.Add(new YamlOutputFormatter());
});
```

Controller:

```
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class WeatherController : ControllerBase
{
    [HttpGet]
    public IActionResult GetWeather() => Ok(new { Temperature = 25, Condition = "Sunny" });
}
```

Test:

  Accept: application/vnd.myapp.v1+json → JSON for v1.
  Accept: application/vnd.myapp.v1+yaml → YAML for v1.

# Best Practices

1. Default Format: Stick to JSON unless clients explicitly request otherwise.
2. Explicit Rejection: Enable ReturnHttpNotAcceptable to return 406 for unsupported formats.
3. Performance: Avoid heavy custom formatters for large payloads; JSON is optimized by default.
4. Documentation: Use Swagger to document supported formats:

```
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "My API", Version = "v1" });
    c.OperationFilter<AddAcceptHeaderOperationFilter>();
});

public class AddAcceptHeaderOperationFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters ??= new List<OpenApiParameter>();
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "Accept",
            In = ParameterLocation.Header,
            Schema = new OpenApiSchema { Type = "string" },
            Description = "Supported: application/json, application/xml, application/x-yaml"
        });
    }
}
```

5. Consistency: Ensure all endpoints support the same formats unless intentionally restricted.

Real-World Scenario

Imagine an e-commerce API:

    Public Clients: Request application/json for lightweight responses.
    Legacy Systems: Request application/xml for compatibility.
    Internal Tools: Use application/x-yaml for human-readable debugging.
    Response: The same /products endpoint serves all formats based on the Accept header.

Sample response for GET /weather:

 JSON: 
 
 ```
    {"temperature": 25, "condition": "Sunny"}
 ```
 
XML: 
```
<WeatherData>
  <Temperature>25</Temperature>
  <Condition>Sunny</Condition>
</WeatherData>
```

YAML:

```
Temperature: 25
Condition: Sunny
```









