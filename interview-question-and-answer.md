I'll provide answers to the advanced .NET Web API interview questions:

## Architecture and Design

### 1. Differences between ASP.NET Core Minimal APIs and Controller-based APIs

**Controller-based APIs:**
- Use the traditional MVC pattern with dedicated controller classes
- Provide robust infrastructure for more complex scenarios
- Include built-in model validation, filters, and dependency injection
- Better for larger applications with many endpoints and complex logic

**Minimal APIs:**
- Introduced in .NET 6, significantly enhanced in .NET 7 and 8
- Allow defining endpoints directly in Program.cs without controllers
- Use a more concise, lambda-based syntax
- Offer lower overhead and faster throughput
- Better for smaller applications or microservices with straightforward requirements
- Example:
```csharp
var app = WebApplication.Create(args);
app.MapGet("/api/products", () => new[] { /* products */ });
app.Run();
```

### 2. Implementing versioning in Web API

**URI Versioning:**
- Embed version in the URI path: `/api/v1/products`
- Pros: Simple, explicit, works with all clients
- Cons: Can't evolve resources independently, potentially cluttered URIs

**Query String Versioning:**
- Add version as a query parameter: `/api/products?api-version=1.0`
- Pros: Cleaner URIs, backward compatible
- Cons: Easy to miss in documentation

**Header Versioning:**
- Pass version in custom HTTP header: `X-API-Version: 1.0`
- Pros: Separates versioning concerns from resources
- Cons: Less discoverable, more complex client code

**Media Type Versioning:**
- Use Accept header with versioned media type: `Accept: application/vnd.company.api.v1+json`
- Pros: Most RESTful approach, tied to representation
- Cons: Most complex to implement and consume

Implementation with Microsoft.AspNetCore.Mvc.Versioning:
```csharp
services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new QueryStringApiVersionReader("api-version"),
        new HeaderApiVersionReader("X-API-Version")
    );
});
```

### 3. Clean Architecture in Web API

Clean Architecture organizes a Web API into concentric layers:

1. **Domain Layer (Core):**
   - Contains business entities, interfaces, domain events
   - Has no dependencies on other layers or external frameworks

2. **Application Layer:**
   - Contains business logic, use cases, commands/queries
   - Depends only on the Domain layer
   - Often uses CQRS pattern with MediatR

3. **Infrastructure Layer:**
   - Implements interfaces defined in inner layers
   - Contains database access, external service integration, etc.
   - Depends on Application and Domain layers

4. **Presentation Layer (API):**
   - Contains controllers, filters, DTOs, serialization
   - Depends on Application layer for business logic

Solution structure:
```
MyApi.Domain
MyApi.Application
MyApi.Infrastructure
MyApi.API
```

Benefits: separation of concerns, testability, maintainability, and independence from frameworks.

### 4. CQRS Pattern in Web API

CQRS (Command Query Responsibility Segregation) separates read and write operations:

- **Commands:** Modify state, don't return data, represent intent (e.g., CreateOrderCommand)
- **Queries:** Return data, don't modify state (e.g., GetOrderByIdQuery)

Implementation with MediatR:

```csharp
// Command
public class CreateProductCommand : IRequest<int>
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Command handler
public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, int>
{
    private readonly IProductRepository _repository;
    
    public CreateProductCommandHandler(IProductRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<int> Handle(CreateProductCommand request, CancellationToken cancellationToken)
    {
        var product = new Product { Name = request.Name, Price = request.Price };
        await _repository.AddAsync(product);
        return product.Id;
    }
}

// Controller
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;
    
    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }
    
    [HttpPost]
    public async Task<ActionResult<int>> Create(CreateProductCommand command)
    {
        return await _mediator.Send(command);
    }
}
```

Benefits: scalability, performance optimization, security separation.

### 5. Mediator Pattern with MediatR

The Mediator pattern decouples communication between components through a central mediator.

Implementation:

1. Install packages:
```
Microsoft.Extensions.DependencyInjection
MediatR
MediatR.Extensions.Microsoft.DependencyInjection
```

2. Configure services:
```csharp
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Startup).Assembly));
```

3. Define requests and handlers:
```csharp
// Query
public class GetProductByIdQuery : IRequest<ProductDto>
{
    public int Id { get; set; }
}

// Query handler
public class GetProductByIdQueryHandler : IRequestHandler<GetProductByIdQuery, ProductDto>
{
    private readonly IProductRepository _repository;
    
    public GetProductByIdQueryHandler(IProductRepository repository)
    {
        _repository = repository;
    }
    
    public async Task<ProductDto> Handle(GetProductByIdQuery request, CancellationToken cancellationToken)
    {
        var product = await _repository.GetByIdAsync(request.Id);
        return new ProductDto { Id = product.Id, Name = product.Name };
    }
}
```

4. Use in controllers:
```csharp
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> Get(int id)
{
    var result = await _mediator.Send(new GetProductByIdQuery { Id = id });
    return result != null ? Ok(result) : NotFound();
}
```

Benefits: simplified controllers, centralized request handling, cross-cutting concerns.

## Performance and Scalability

### 6. Strategies to improve Web API performance

1. **Asynchronous programming:**
   - Use async/await for I/O operations
   - Avoid blocking threads with `Task.Result` or `.Wait()`

2. **Response compression:**
   ```csharp
   services.AddResponseCompression();
   ```

3. **Data caching:**
   - Implement various caching strategies
   - Use ETags for conditional requests

4. **Efficient data access:**
   - Use database indexes
   - Implement projection queries (select only needed fields)
   - Use compiled LINQ queries

5. **Minimize serialization overhead:**
   - Use DTOs to return only needed data
   - Consider faster serializers like System.Text.Json

6. **Connection pooling:**
   - Reuse database connections
   - Configure appropriate pool sizes

7. **Load balancing and horizontal scaling**

8. **Optimize startup:**
   - Use minimal hosting model
   - Consider Ahead-of-Time (AOT) compilation

9. **Use pagination for large datasets**

10. **Consider NoSQL where appropriate**

### 7. Caching in Web API

1. **Response Caching:**
   ```csharp
   // In Startup.ConfigureServices
   services.AddResponseCaching();
   
   // In Startup.Configure
   app.UseResponseCaching();
   
   // In Controller
   [HttpGet]
   [ResponseCache(Duration = 60)]
   public IActionResult Get() { ... }
   ```

2. **Memory Cache:**
   ```csharp
   // In Startup.ConfigureServices
   services.AddMemoryCache();
   
   // Usage
   private readonly IMemoryCache _cache;
   
   public ProductsController(IMemoryCache cache)
   {
       _cache = cache;
   }
   
   [HttpGet("{id}")]
   public async Task<IActionResult> Get(int id)
   {
       if (!_cache.TryGetValue($"product_{id}", out Product product))
       {
           product = await _repository.GetByIdAsync(id);
           _cache.Set($"product_{id}", product, TimeSpan.FromMinutes(10));
       }
       return Ok(product);
   }
   ```

3. **Distributed Cache:**
   ```csharp
   // In Startup.ConfigureServices
   services.AddStackExchangeRedisCache(options =>
   {
       options.Configuration = "localhost:6379";
       options.InstanceName = "SampleInstance";
   });
   
   // Usage
   private readonly IDistributedCache _cache;
   
   [HttpGet("{id}")]
   public async Task<IActionResult> Get(int id)
   {
       var cacheKey = $"product_{id}";
       var jsonProduct = await _cache.GetStringAsync(cacheKey);
       
       if (jsonProduct == null)
       {
           var product = await _repository.GetByIdAsync(id);
           jsonProduct = JsonSerializer.Serialize(product);
           await _cache.SetStringAsync(cacheKey, jsonProduct, new DistributedCacheEntryOptions
           {
               AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
           });
       }
       
       return Ok(JsonSerializer.Deserialize<Product>(jsonProduct));
   }
   ```

### 8. Rate limiting in ASP.NET Core Web API

In .NET 7/8, use the built-in rate limiting middleware:

```csharp
// In Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
    {
        return RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Connection.RemoteIpAddress?.ToString() ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                AutoReplenishment = true,
                PermitLimit = 100,
                QueueLimit = 0,
                Window = TimeSpan.FromMinutes(1)
            });
    });
    
    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsync("Too many requests. Please try again later.", token);
    };
});

// In Program.cs Configure
app.UseRateLimiter();

// Per-endpoint rate limiting
app.MapGet("/api/limited", async (HttpContext context) =>
{
    await context.Response.WriteAsync("Rate limited endpoint");
})
.RequireRateLimiting("fixed");
```

For earlier versions, use third-party packages like AspNetCoreRateLimit.

### 9. Background processing with IHostedService and BackgroundService

**BackgroundService** is an abstract base class that implements IHostedService:

```csharp
public class QueueProcessorService : BackgroundService
{
    private readonly ILogger<QueueProcessorService> _logger;
    private readonly IServiceProvider _serviceProvider;
    
    public QueueProcessorService(ILogger<QueueProcessorService> logger, IServiceProvider serviceProvider)
    {
        _logger = logger;
        _serviceProvider = serviceProvider;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Queue Processing Service is starting.");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            // Process items from a queue
            using (var scope = _serviceProvider.CreateScope())
            {
                var processor = scope.ServiceProvider.GetRequiredService<IQueueProcessor>();
                await processor.ProcessQueueAsync(stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
        
        _logger.LogInformation("Queue Processing Service is stopping.");
    }
}

// Register in Program.cs
services.AddHostedService<QueueProcessorService>();
```

For more complex scenarios, consider using:
- Quartz.NET for scheduled jobs
- Hangfire for persistent background processing
- Worker Service template for standalone workers

### 10. Health checks in ASP.NET Core Web API

Health checks monitor app status and dependencies:

```csharp
// In Program.cs
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<ApplicationDbContext>()
    .AddRedis(Configuration["RedisConnection"])
    .AddCheck<CustomHealthCheck>("custom");

// In Program.cs Configure
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

// Custom health check
public class CustomHealthCheck : IHealthCheck
{
    private readonly IHttpClientFactory _httpClientFactory;
    
    public CustomHealthCheck(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
    }
    
    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        try
        {
            var client = _httpClientFactory.CreateClient();
            var response = await client.GetAsync("https://external-service.com/health");
            
            return response.IsSuccessStatusCode 
                ? HealthCheckResult.Healthy("External service is available")
                : HealthCheckResult.Degraded("External service returned non-success status code");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("External service check failed", ex);
        }
    }
}
```

## Security

### 11. Authentication mechanisms in ASP.NET Core Web API

**JWT Authentication:**
```csharp
// In Program.cs
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = Configuration["Jwt:Issuer"],
            ValidAudience = Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
        };
    });

// Token generation
public string GenerateJwtToken(User user)
{
    var claims = new[]
    {
        new Claim(JwtRegisteredClaimNames.Sub, user.Email),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
        new Claim(ClaimTypes.NameIdentifier, user.Id)
    };
    
    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
    
    var token = new JwtSecurityToken(
        issuer: _configuration["Jwt:Issuer"],
        audience: _configuration["Jwt:Audience"],
        claims: claims,
        expires: DateTime.Now.AddMinutes(30),
        signingCredentials: creds);
    
    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

**OAuth 2.0 & OpenID Connect:**
```csharp
services.AddAuthentication(options =>
{
    options.DefaultScheme = "Cookies";
    options.DefaultChallengeScheme = "oidc";
})
.AddCookie("Cookies")
.AddOpenIdConnect("oidc", options =>
{
    options.Authority = "https://identity-provider.com";
    options.ClientId = "web-api";
    options.ClientSecret = "secret";
    options.ResponseType = "code";
    options.SaveTokens = true;
    options.Scope.Add("api1");
    options.Scope.Add("offline_access");
});
```

**Identity Server:**
Used as an OAuth 2.0/OpenID Connect provider:
- Creates and validates tokens
- Manages identity and access control
- Supports multiple clients and scopes

### 12. API key authentication in a Web API

Custom implementation:

```csharp
// API key authentication handler
public class ApiKeyAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    private const string ApiKeyHeaderName = "X-API-Key";
    private readonly IApiKeyValidator _apiKeyValidator;
    
    public ApiKeyAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock,
        IApiKeyValidator apiKeyValidator)
        : base(options, logger, encoder, clock)
    {
        _apiKeyValidator = apiKeyValidator;
    }
    
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(ApiKeyHeaderName, out var apiKeyHeaderValues))
        {
            return AuthenticateResult.Fail("API Key was not provided.");
        }
        
        var providedApiKey = apiKeyHeaderValues.FirstOrDefault();
        
        if (apiKeyHeaderValues.Count == 0 || string.IsNullOrWhiteSpace(providedApiKey))
        {
            return AuthenticateResult.Fail("API Key was not provided.");
        }
        
        if (!await _apiKeyValidator.IsValidApiKeyAsync(providedApiKey))
        {
            return AuthenticateResult.Fail("Invalid API Key provided.");
        }
        
        var claims = new[] {
            new Claim(ClaimTypes.Name, "API_User"),
            // Add additional claims based on the API key
        };
        
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);
        
        return AuthenticateResult.Success(ticket);
    }
}

// Register in Program.cs
services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = "ApiKey";
    options.DefaultChallengeScheme = "ApiKey";
})
.AddScheme<AuthenticationSchemeOptions, ApiKeyAuthHandler>("ApiKey", null);
```

### 13. Securing sensitive data and secrets management

1. **User Secrets (Development)**:
```csharp
// In .csproj
<PropertyGroup>
  <UserSecretsId>aspnet-WebApi-E3E85F57-7A5D-4354-A372-4F9297FE6380</UserSecretsId>
</PropertyGroup>

// Access
var connectionString = Configuration["ConnectionStrings:DefaultConnection"];
```

2. **Environment Variables**:
```csharp
// AppSettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=..."
  }
}

// Override with environment variables
// In Windows: setx ASPNETCORE_ENVIRONMENT "Production"
// In Windows: setx ConnectionStrings__DefaultConnection "Server=prod-server;..."
// In Linux: export ConnectionStrings__DefaultConnection="Server=prod-server;..."
```

3. **Azure Key Vault**:
```csharp
// In Program.cs
var keyVaultEndpoint = new Uri(Environment.GetEnvironmentVariable("VaultUri"));
builder.Configuration.AddAzureKeyVault(keyVaultEndpoint, new DefaultAzureCredential());
```

4. **Encrypted configuration sections**:
```csharp
services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo(@"c:\keys\"))
    .ProtectKeysWithCertificate(new X509Certificate2("certificate.pfx", "password"));
```

5. **Managed Identity for Azure resources**

6. **Secret rotation and management**

### 14. CORS in ASP.NET Core Web API

CORS (Cross-Origin Resource Sharing) allows restricted resources to be requested from another domain.

Secure implementation:

```csharp
// In Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("ProductionPolicy", builder =>
    {
        builder.WithOrigins("https://trusted-app.com", "https://admin.trusted-app.com")
               .WithMethods("GET", "POST", "PUT", "DELETE")
               .WithHeaders("Authorization", "Content-Type")
               .AllowCredentials()
               .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
    
    options.AddPolicy("DevelopmentPolicy", builder =>
    {
        builder.AllowAnyOrigin()
               .AllowAnyMethod()
               .AllowAnyHeader();
    });
});

// In Configure middleware
if (app.Environment.IsDevelopment())
{
    app.UseCors("DevelopmentPolicy");
}
else
{
    app.UseCors("ProductionPolicy");
}

// Per-endpoint CORS (alternative approach)
app.MapGet("/api/public", () => new[] { "public data" })
   .RequireCors("PublicPolicy");
```

### 15. Role-based and policy-based authorization

**Role-based authorization**:
```csharp
// In Program.cs
services.AddAuthorization();

// In controller
[Authorize(Roles = "Admin,Manager")]
public class AdminController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Admin data");
    
    [Authorize(Roles = "Admin")]
    [HttpPost]
    public IActionResult Post() => Ok("Admin-only action");
}
```

**Policy-based authorization**:
```csharp
// In Program.cs
services.AddAuthorization(options =>
{
    // Simple policy
    options.AddPolicy("RequireAdminRole", policy => policy.RequireRole("Admin"));
    
    // Claim-based policy
    options.AddPolicy("AtLeast21", policy => 
        policy.RequireAssertion(context => 
            context.User.HasClaim(c => c.Type == "DateOfBirth") &&
            DateTime.Parse(context.User.FindFirst(c => c.Type == "DateOfBirth").Value) <= DateTime.Now.AddYears(-21)
        ));
    
    // Multiple requirements
    options.AddPolicy("ResourceAccess", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("Permission", "EditResource");
        policy.RequireRole("Editor");
    });
});

// Custom policy with requirements
services.AddAuthorization(options =>
{
    options.AddPolicy("ResourceOwner", policy =>
        policy.Requirements.Add(new ResourceOwnerRequirement()));
});
services.AddSingleton<IAuthorizationHandler, ResourceOwnerAuthorizationHandler>();

// Usage in controller
[Authorize(Policy = "ResourceOwner")]
[HttpPut("resources/{id}")]
public IActionResult Update(int id, ResourceDto resource) { ... }

// Custom requirement and handler
public class ResourceOwnerRequirement : IAuthorizationRequirement { }

public class ResourceOwnerAuthorizationHandler 
    : AuthorizationHandler<ResourceOwnerRequirement, ResourceDto>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ResourceOwnerRequirement requirement,
        ResourceDto resource)
    {
        if (context.User.FindFirstValue(ClaimTypes.NameIdentifier) == resource.OwnerId)
        {
            context.Succeed(requirement);
        }
        
        return Task.CompletedTask;
    }
}
```

## API Design and Documentation

### 16. Documenting Web APIs using Swagger/OpenAPI

```csharp
// In Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "An API to perform operations",
        TermsOfService = new Uri("https://example.com/terms"),
        Contact = new OpenApiContact
        {
            Name = "John Doe",
            Email = "john@example.com",
            Url = new Uri("https://twitter.com/johndoe"),
        },
        License = new OpenApiLicense
        {
            Name = "API License",
            Url = new Uri("https://example.com/license"),
        }
    });
    
    // Add JWT authentication
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme.",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });
    
    c.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
    
    // Include XML comments
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    c.IncludeXmlComments(xmlPath);
    
    // Enable annotations
    c.EnableAnnotations();
});

// In Program.cs Configure
app.UseSwagger();
app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "My API V1");
    c.RoutePrefix = string.Empty; // Set Swagger UI at root
    c.DefaultModelsExpandDepth(-1); // Hide schemas section
    c.DocExpansion(DocExpansion.None); // Collapse operations by default
});

// Document controllers and actions
/// <summary>
/// Manages products in the catalog
/// </summary>
[SwaggerTag("Products management")]
[Produces("application/json")]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    /// <summary>
    /// Retrieves all products
    /// </summary>
    /// <remarks>
    /// Sample request:
    ///     GET /api/products
    /// </remarks>
    /// <returns>A list of products</returns>
    /// <response code="200">Returns the products list</response>
    /// <response code="401">If the user is not authenticated</response>
    [HttpGet]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
    {
        // ...
    }
}
```

### 17. RESTful API design best practices

1. **Use proper HTTP methods:**
   - GET: Read resources
   - POST: Create resources
   - PUT: Update resources (full update)
   - PATCH: Partial update
   - DELETE: Remove resources

2. **Use nouns, not verbs in endpoints:**
   - Good: `/api/products`
   - Bad: `/api/getProducts`

3. **Use plurals for collection resources:**
   - `/api/products` (collection)
   - `/api/products/123` (specific item)

4. **Use nested resources for relationships:**
   - `/api/orders/123/items`

5. **Use proper HTTP status codes:**
   - 200 OK: Successful request
   - 201 Created: Resource created
   - 204 No Content: Successful but no response body
   - 400 Bad Request: Invalid input
   - 401 Unauthorized: Authentication required
   - 403 Forbidden: Authenticated but not authorized
   - 404 Not Found: Resource not found
   - 409 Conflict: Resource state conflict
   - 422 Unprocessable Entity: Validation errors
   - 500 Internal Server Error: Server error

6. **Use filtering, sorting, paging:**
   - `/api/products?category=electronics&sort=price&page=2&pageSize=10`

7. **Versioning your API:**
   - `/api/v1/products`

8. **Use problem details for errors (RFC 7807):**
```json
{
  "type": "https://example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "The product request is invalid",
  "instance": "/api/products/validation-error/12345",
  "errors": {
    "Name": ["Name is required"],
    "Price": ["Price must be greater than zero"]
  }
}
```

9. **Implement idempotent operations:**
   - PUT and DELETE should be idempotent

10. **Support content negotiation:**
    - Use Accept and Content-Type headers

### 18. Implementing HATEOAS in Web API

HATEOAS (Hypermedia as the Engine of Application State) adds hypermedia links to responses:

```csharp
// Link model
public class Link
{
    public string Href { get; set; }
    public string Rel { get; set; }
    public string Method { get; set; }
    
    public Link(string href, string rel, string method)
    {
        Href = href;
        Rel = rel;
        Method = method;
    }
}

// Resource with links
public class ProductResource
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public List<Link> Links { get; set; } = new List<Link>();
}

// In controller
[HttpGet("{id}")]
public async Task<ActionResult<ProductResource>> Get(int id)
{
    var product = await _repository.GetByIdAsync(id);
    
    if (product == null)
        return NotFound();
    
    var resource = new ProductResource
    {
        Id = product.Id,
        Name = product.Name,
        Price = product.Price
    };
    
    // Add links
    resource.Links.Add(
        new Link(
            Url.Link("GetProduct", new { id = product.Id }),
            "self",
            "GET"));
            
    resource.Links.Add(
        new Link(
            Url.Link("UpdateProduct", new { id = product.Id }),
            "update_product",
            "PUT"));
            
    resource.Links.Add(
        new Link(
            Url.Link("DeleteProduct", new { id = product.Id }),
            "delete_product",
            "DELETE"));
            
    return resource;
}
```

For an automated approach, consider using libraries like:
- HAL (Hypertext Application Language)
- ASP.NET Core HATEOAS extensions

### 19. Differences between OData and GraphQL

**OData:**
- Microsoft standard for building RESTful APIs
- Uses URL query parameters for filtering, sorting, and selecting
- Server defines the data model and query capabilities
- Implementation:

```csharp
// In Program.cs
builder.Services.AddControllers().AddOData(opt => 
    opt.Select().Filter().Expand().OrderBy().SetMaxTop(100).Count());

// In Controller
[HttpGet]
[EnableQuery]
public IActionResult Get()
{
    return Ok(_context.Products);
}

// Example queries:
// /api/products?$filter=Category eq 'Electronics'
// /api/
