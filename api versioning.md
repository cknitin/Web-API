# API Versioning


## Common Versioning Strategies

There are several widely adopted strategies for API versioning. Each has its pros and cons, depending on your use case:
1. URI Path Versioning (Most Common)

    Description: The version is included in the URL path 
    ```
    (e.g., /api/v1/resource).
    ```
    - Pros:
        - Simple, intuitive, highly discoverable via URLs.
    - Cons:
        - Can clutter the URL structure over time; requires routing adjustments.
    
    Example: 
    ```
    GET /api/v1/users vs. GET /api/v2/users.
    ```

3. Query String Versioning

    Description: The version is passed as a query parameter (e.g., /api/users?api-version=1.0).
    - Pros:
        - Clean URLs, flexible for optional versioning.
    - Cons:
         - Less discoverable, might conflict with other query parameters.
    
    Example: 
    ```
    GET /api/users?api-version=1.0.
    ```

5. Header Versioning

    Description: The version is specified in a custom HTTP header (e.g., Api-Version: 1.0).
    - Pros:
        - Keeps URLs clean, aligns with REST principles.
    - Cons:
        - Less visible to clients, requires client-side support for headers.
    
    ```
        Example: GET /api/users with header Api-Version: 1.0.
    ```

6. Media Type Versioning (Content Negotiation)

    Description: The version is embedded in the Accept header’s media type (e.g., Accept: application/vnd.myapp.v1+json).
    - Pros:
        - RESTful, leverages HTTP standards, supports content negotiation.
     - Cons:
         - Complex for clients to implement, less intuitive.
    
   ``` 
       Example: GET /api/users with Accept: application/vnd.myapp.v1+json.
   ```

## Step 1: Install the Package

```
dotnet add package Microsoft.AspNetCore.Mvc.Versioning
```

## Step 2: Configure Versioning in Startup.cs or Program.cs

```
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0); // Default version if none specified
    options.AssumeDefaultVersionWhenUnspecified = true; // Use default if no version provided
    options.ReportApiVersions = true; // Include supported versions in response headers
    options.ApiVersionReader = new UrlSegmentApiVersionReader(); // Read version from URL
});

var app = builder.Build();

// Configure the HTTP request pipeline
app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Explanation:
- DefaultApiVersion: Sets v1.0 as the default version.
- AssumeDefaultVersionWhenUnspecified: Ensures unversioned requests (e.g., /api/users) use v1.0.
- ReportApiVersions: Adds an api-supported-versions header to responses.
- UrlSegmentApiVersionReader: Reads the version from the URL path (e.g., /api/v1/users).


## Step 3: Create Versioned Controllers

Define controllers with specific versions using the [ApiVersion] attribute.

- Version 1.0 Controller:

```
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new[] { "User1 v1", "User2 v1" });
    }
}
```

- Version 2.0 Controller:

```
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersControllerV2 : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new[] { "User1 v2", "User2 v2", "User3 v2" }); // Updated response
    }
}
```

## Step 4: Test the API

    Request v1: GET /api/v1/users
        Response: ["User1 v1", "User2 v1"]
        Headers: api-supported-versions: 1.0, 2.0
    Request v2: GET /api/v2/users
        Response: ["User1 v2", "User2 v2", "User3 v2"]
    Unversioned Request: GET /api/users
        Response: ["User1 v1", "User2 v1"] (default v1.0)

# Advanced Features and Considerations

## 1. Deprecating Versions

Mark older versions as deprecated to notify clients:

```
[ApiVersion("1.0", Deprecated = true)]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiController]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Deprecated v1");
}
```

- Clients receive a warning in the response header: api-deprecated-versions: 1.0.

## 2. Multiple Version Readers

Combine versioning strategies (e.g., URL and header):

```
options.ApiVersionReader = ApiVersionReader.Combine(
    new UrlSegmentApiVersionReader(),
    new HeaderApiVersionReader("x-api-version")
);
```

- Supports both /api/v1/users and /api/users with header x-api-version: 1.0.

3. Versioning by Namespace

Organize controllers by version in different namespaces:

```
namespace MyApi.Controllers.V1
{
    [ApiVersion("1.0")]
    [Route("api/[controller]")]
    public class UsersController : ControllerBase { ... }
}

namespace MyApi.Controllers.V2
{
    [ApiVersion("2.0")]
    [Route("api/[controller]")]
    public class UsersController : ControllerBase { ... }
}
```

Configure: options.Conventions.Controller<UsersController>().HasApiVersion(new ApiVersion(1, 0));.

## 4. Semantic Versioning

Use major.minor versioning (e.g., 1.0, 1.1, 2.0):

    Major: Breaking changes.
    Minor: Backward-compatible changes.

## 5. Swagger Integration

Integrate with Swagger for versioned documentation:

```
services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "API v1", Version = "v1" });
    c.SwaggerDoc("v2", new OpenApiInfo { Title = "API v2", Version = "v2" });
    c.EnableAnnotations();
});
```
