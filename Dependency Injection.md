# Dependency Injection

Here's a detailed explanation and implementation of **Dependency Injection (DI)** in a .NET Core Web API, focusing on the latest versions (e.g., .NET 6, 7, or 8 as of February 27, 2025). I'll cover what DI is, why it's beneficial, and how to use it effectively in a Web API, with examples and advanced scenarios.

---

### What is Dependency Injection?

**Dependency Injection** is a design pattern and a core feature of ASP.NET Core that allows you to provide (or "inject") dependencies (e.g., services, repositories) into a class rather than having the class create them itself. It promotes loose coupling, testability, and maintainability by inverting control of object creation to a container.

- **Without DI**: A class creates its own dependencies (tight coupling).
- **With DI**: Dependencies are provided externally via constructor, method, or property injection.

In .NET Core, DI is built into the framework via the `IServiceProvider` and `IServiceCollection`, making it seamless for Web APIs.

---

### Why Use Dependency Injection?

1. **Loose Coupling**: Classes depend on abstractions (interfaces) rather than concrete implementations, making it easy to swap implementations (e.g., for testing or upgrades).
2. **Testability**: Inject mocks or stubs during unit tests instead of real dependencies.
3. **Maintainability**: Centralizes dependency configuration, reducing code duplication and improving readability.
4. **Scalability**: Simplifies managing complex dependency graphs in large applications.

#### Real-World Example
Imagine an e-commerce API needing a payment service:
- Without DI, a controller creates a `PaymentService` directly, hardcoding its dependency.
- With DI, the `PaymentService` is injected, allowing you to switch between `StripePaymentService` and `PayPalPaymentService` without changing the controller.

---

### How Dependency Injection Works in .NET Core

#### Key Components
- **`IServiceCollection`**: Registers services and their lifetimes.
- **`IServiceProvider`**: Resolves and provides instances of registered services.
- **Service Lifetimes**:
  - **Transient**: New instance per request.
  - **Scoped**: Same instance per HTTP request.
  - **Singleton**: Same instance for the app’s lifetime.

---

### Implementation in .NET 8 Web API

#### Step 1: Basic Setup
In `Program.cs` (minimal hosting model):

```csharp
using Microsoft.AspNetCore.Mvc;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Register dependencies
builder.Services.AddScoped<IPaymentService, StripePaymentService>(); // Scoped service
builder.Services.AddTransient<ILoggerAdapter, ConsoleLoggerAdapter>(); // Transient service
builder.Services.AddSingleton<IConfigProvider, AppConfigProvider>(); // Singleton service

var app = builder.Build();

app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

- **Explanation**:
  - `AddScoped`: `IPaymentService` gets a new instance per HTTP request.
  - `AddTransient`: `ILoggerAdapter` gets a new instance every time it’s resolved.
  - `AddSingleton`: `IConfigProvider` is created once and shared across the app.

#### Step 2: Define Dependencies
```csharp
public interface IPaymentService
{
    Task ProcessPaymentAsync(decimal amount);
}

public class StripePaymentService : IPaymentService
{
    public Task ProcessPaymentAsync(decimal amount)
    {
        // Simulate payment processing
        Console.WriteLine($"Processed payment of {amount:C} via Stripe");
        return Task.CompletedTask;
    }
}

public interface ILoggerAdapter
{
    void Log(string message);
}

public class ConsoleLoggerAdapter : ILoggerAdapter
{
    public void Log(string message) => Console.WriteLine(message);
}

public interface IConfigProvider
{
    string GetConfig(string key);
}

public class AppConfigProvider : IConfigProvider
{
    public string GetConfig(string key) => $"Config value for {key}";
}
```

#### Step 3: Use DI in a Controller
```csharp
[ApiController]
[Route("[controller]")]
public class PaymentController : ControllerBase
{
    private readonly IPaymentService _paymentService;
    private readonly ILoggerAdapter _logger;
    private readonly IConfigProvider _config;

    // Constructor injection
    public PaymentController(IPaymentService paymentService, ILoggerAdapter logger, IConfigProvider config)
    {
        _paymentService = paymentService;
        _logger = logger;
        _config = config;
    }

    [HttpPost]
    public async Task<IActionResult> ProcessPayment([FromBody] decimal amount)
    {
        _logger.Log($"Processing payment of {amount:C}");
        await _paymentService.ProcessPaymentAsync(amount);
        var apiKey = _config.GetConfig("ApiKey");
        _logger.Log($"Using config: {apiKey}");
        return Ok(new { Status = "Payment processed" });
    }
}
```

- **Explanation**:
  - Dependencies (`IPaymentService`, `ILoggerAdapter`, `IConfigProvider`) are injected via the constructor.
  - The `IServiceProvider` resolves them based on their registered lifetimes.

#### Step 4: Test the API
- **Request**: `POST /payment` with body `100.50`
- **Output**:
  ```
  Processing payment of $100.50
  Processed payment of $100.50 via Stripe
  Using config: Config value for ApiKey
  ```
- **Response**: `{"Status": "Payment processed"}`

---

### Advanced Dependency Injection Features

#### 1. **Factory Registration**
For dynamic instantiation:

```csharp
builder.Services.AddScoped<IPaymentService>(serviceProvider =>
{
    var config = serviceProvider.GetRequiredService<IConfigProvider>();
    return config.GetConfig("PaymentProvider") == "PayPal" 
        ? new PayPalPaymentService() 
        : new StripePaymentService();
});
```

- **Explanation**: Chooses `PayPalPaymentService` or `StripePaymentService` based on runtime config.

#### 2. **Keyed Services (.NET 8)**
Register and resolve services by a key:

```csharp
builder.Services.AddKeyedScoped<IPaymentService, StripePaymentService>("stripe");
builder.Services.AddKeyedScoped<IPaymentService, PayPalPaymentService>("paypal");

// Controller
public PaymentController([FromKeyedServices("stripe")] IPaymentService stripeService)
{
    _stripeService = stripeService;
}
```

- **Explanation**: Allows multiple implementations of the same interface, resolved by key.

#### 3. **Method Injection**
Inject services into specific methods:

```csharp
[HttpGet]
public IActionResult GetConfig([FromServices] IConfigProvider config)
{
    return Ok(config.GetConfig("ApiKey"));
}
```

- **Explanation**: Useful for occasional dependencies without cluttering the constructor.

#### 4. **Custom Lifetime Management**
Create a custom scope:

```csharp
[HttpPost("batch")]
public async Task<IActionResult> BatchProcess([FromServices] IServiceProvider serviceProvider)
{
    using (var scope = serviceProvider.CreateScope())
    {
        var paymentService = scope.ServiceProvider.GetRequiredService<IPaymentService>();
        await paymentService.ProcessPaymentAsync(50m);
    }
    return Ok("Batch processed");
}
```

- **Explanation**: Ensures scoped services (e.g., DB contexts) are disposed properly within a custom scope.

#### 5. **Middleware with DI**
Inject dependencies into middleware:

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILoggerAdapter _logger;

    public LoggingMiddleware(RequestDelegate next, ILoggerAdapter logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.Log($"Request: {context.Request.Path}");
        await _next(context);
    }
}

// Register in Program.cs
app.UseMiddleware<LoggingMiddleware>();
```

---

### Best Practices

1. **Use Interfaces**: Depend on abstractions (`IPaymentService`) rather than concrete classes (`StripePaymentService`).
2. **Choose Lifetimes Wisely**:
   - **Transient**: For lightweight, stateless services.
   - **Scoped**: For services tied to a request (e.g., DB contexts).
   - **Singleton**: For shared, thread-safe services (e.g., config).
3. **Avoid Service Locator**: Prefer constructor injection over `IServiceProvider.GetService` to maintain clarity.
4. **Validate Dependencies**: Ensure required services are registered to avoid runtime errors:
   ```csharp
   builder.Services.AddControllers()
       .ConfigureApiBehaviorOptions(options => options.SuppressModelStateInvalidFilter = true);
   ```
5. **Testability**: Use DI to inject mocks in unit tests:
   ```csharp
   var mockPayment = new Mock<IPaymentService>();
   var controller = new PaymentController(mockPayment.Object, new ConsoleLoggerAdapter(), new AppConfigProvider());
   ```

---

### Real-World Scenario
In an e-commerce API:
- **Payment Processing**: `IPaymentService` (scoped) switches between Stripe and PayPal based on config.
- **Logging**: `ILoggerAdapter` (transient) logs each step without reusing instances.
- **Configuration**: `IConfigProvider` (singleton) provides app-wide settings.

#### Sample Flow
- User submits payment → `PaymentController` injects `IPaymentService` (Stripe), `ILoggerAdapter`, and `IConfigProvider`.
- Stripe processes payment, logs actions, and uses config → Response sent back.

---

### Why It Improves Web APIs
- **Flexibility**: Swap payment providers without changing controller code.
- **Scalability**: Proper lifetimes ensure resource efficiency.
- **Maintenance**: Centralized service registration simplifies updates.

Dependency Injection in .NET 8 Web API is a foundational feature that enhances design quality and flexibility. Let me know if you’d like to explore a specific DI scenario (e.g., testing, custom containers) further!
