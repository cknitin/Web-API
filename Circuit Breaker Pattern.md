# Circuit Breaker Pattern

Here's a detailed explanation and implementation of the **Circuit Breaker Pattern** in a .NET Core Web API, focusing on the latest versions (e.g., .NET 6, 7, or 8 as of February 27, 2025). I'll use **Polly**, a popular .NET resilience library, to demonstrate this pattern in a real-world scenario.

---

### What is the Circuit Breaker Pattern?

The Circuit Breaker Pattern is a design pattern used to prevent an application from repeatedly attempting to execute an operation that’s likely to fail (e.g., calling an unreliable external service). It acts like an electrical circuit breaker: when failures exceed a threshold, it "trips" (opens), stopping further calls for a period, then attempts to "reset" (half-open) to test if the issue is resolved.

#### States
1. **Closed**: Normal operation; requests are allowed.
2. **Open**: Failure threshold reached; requests are blocked for a cooldown period.
3. **Half-Open**: After cooldown, a limited number of test requests are allowed to check recovery.

#### Why Use It?
- **Resilience**: Prevents cascading failures in distributed systems.
- **Performance**: Avoids wasting resources on doomed calls.
- **Recovery**: Automatically retries when the service might be back online.

---

### Real-World Scenario: E-Commerce API

Imagine an e-commerce Web API that calls an external **Payment Gateway API**. The payment service occasionally fails due to network issues or overload. Without a circuit breaker, the API would keep retrying, slowing down or crashing under load. With a circuit breaker:
- After a few failures, it stops calling the payment service (open state).
- After a cooldown, it tests the service (half-open).
- If successful, it resumes normal operation (closed).

---

### Implementation in .NET 8 Web API with Polly

#### Step 1: Install Polly
```bash
dotnet add package Polly
```

#### Step 2: Configure Circuit Breaker
In `Program.cs`:

```csharp
using Microsoft.AspNetCore.Builder;
using Polly;
using Polly.CircuitBreaker;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddHttpClient("PaymentGateway", client =>
{
    client.BaseAddress = new Uri("https://paymentgateway.example.com");
    client.Timeout = TimeSpan.FromSeconds(10);
})
.AddPolicyHandler(GetCircuitBreakerPolicy());

var app = builder.Build();

app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();

// Define Circuit Breaker Policy
static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return Policy
        .Handle<HttpRequestException>() // Network errors
        .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode) // Non-200 responses
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 3, // Break after 3 failures
            durationOfBreak: TimeSpan.FromSeconds(30), // Stay open for 30 seconds
            onBreak: (result, breakDuration) =>
            {
                Console.WriteLine($"Circuit opened for {breakDuration.TotalSeconds} seconds due to {result.Exception?.Message ?? result.Result.StatusCode}");
            },
            onReset: () => Console.WriteLine("Circuit reset"),
            onHalfOpen: () => Console.WriteLine("Circuit half-open, testing recovery")
        );
}
```

- **Explanation**:
  - **`AddHttpClient`**: Configures an `HttpClient` for the payment gateway.
  - **`CircuitBreakerAsync`**:
    - Breaks after 3 consecutive failures.
    - Stays open for 30 seconds.
    - Logs state changes for monitoring.

#### Step 3: Use Circuit Breaker in Controller
```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyWebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class PaymentController : ControllerBase
    {
        private readonly IHttpClientFactory _httpClientFactory;

        public PaymentController(IHttpClientFactory httpClientFactory)
        {
            _httpClientFactory = httpClientFactory;
        }

        [HttpPost]
        public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request)
        {
            var client = _httpClientFactory.CreateClient("PaymentGateway");

            try
            {
                var response = await client.PostAsJsonAsync("/api/payments", request);
                response.EnsureSuccessStatusCode();
                var result = await response.Content.ReadAsStringAsync();
                return Ok(new { Status = "Success", Details = result });
            }
            catch (BrokenCircuitException ex)
            {
                // Circuit is open
                return StatusCode(503, new { Status = "Service Unavailable", Message = "Payment gateway is temporarily down. Try again later." });
            }
            catch (HttpRequestException ex)
            {
                // Other failures before circuit breaks
                return StatusCode(500, new { Status = "Error", Message = ex.Message });
            }
        }
    }

    public class PaymentRequest
    {
        public decimal Amount { get; set; }
        public string CardNumber { get; set; }
    }
}
```

- **Explanation**:
  - Calls the payment gateway API via the configured `HttpClient`.
  - Handles `BrokenCircuitException` when the circuit is open, returning `503 Service Unavailable`.

#### Step 4: Test the API
- **Normal Operation (Closed)**:
  - `POST /payment` with `{"amount": 100, "cardNumber": "1234-5678-9012-3456"}`
  - Response: `200 OK` with payment details.
- **Failures (Trigger Open)**:
  - If the payment gateway fails 3 times (e.g., timeouts or 500 errors), the circuit opens.
  - Next request → `503 Service Unavailable`.
- **Cooldown (Half-Open)**:
  - After 30 seconds, the next request tests the service.
  - If successful → Circuit closes; if it fails → Circuit stays open longer.

---

### How It Works in the Real World

#### Scenario Breakdown
1. **Normal Shopping**:
   - User adds items to cart and checks out.
   - `POST /payment` succeeds, circuit remains closed.

2. **Payment Gateway Outage**:
   - Gateway fails 3 times (e.g., network issue).
   - Circuit opens for 30 seconds.
   - API returns `503` to clients, suggesting they retry later (e.g., "Payment unavailable, try again in a minute").

3. **Recovery**:
   - After 30 seconds, the circuit goes half-open.
   - Next payment attempt succeeds → Circuit closes, resuming normal operation.
   - If it fails → Circuit reopens for another 30 seconds.

#### Logs
```
Circuit opened for 30 seconds due to Timeout
Circuit half-open, testing recovery
Circuit reset
```

---

### Advanced Circuit Breaker Features

#### 1. Combining with Retry
Add retry logic before breaking:

```csharp
static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    var retry = Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(2, retryAttempt => TimeSpan.FromSeconds(retryAttempt));

    var circuitBreaker = Policy
        .Handle<HttpRequestException>()
        .OrResult<HttpResponseMessage>(r => !r.IsSuccessStatusCode)
        .CircuitBreakerAsync(3, TimeSpan.FromSeconds(30));

    return Policy.WrapAsync(retry, circuitBreaker);
}
```

- **Explanation**: Retries twice per call before counting toward the circuit breaker’s failure threshold.

#### 2. Dynamic Configuration
Adjust based on runtime conditions:

```csharp
builder.Services.AddHttpClient("PaymentGateway")
    .AddPolicyHandler(request => Policy
        .Handle<HttpRequestException>()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: request.RequestUri?.Host.Contains("test") == true ? 5 : 3,
            durationOfBreak: TimeSpan.FromSeconds(30)));
```

- **Explanation**: 5 failures for test environments, 3 for production.

#### 3. Monitoring and Metrics
Integrate with logging or telemetry:

```csharp
circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(3, TimeSpan.FromSeconds(30),
        onBreak: (result, breakDuration) => app.Services.GetRequiredService<ILogger<PaymentController>>()
            .LogWarning("Circuit opened for {Duration}s due to {Reason}", breakDuration.TotalSeconds, result.Exception?.Message),
        onReset: () => app.Services.GetRequiredService<ILogger<PaymentController>>().LogInformation("Circuit reset"),
        onHalfOpen: () => app.Services.GetRequiredService<ILogger<PaymentController>>().LogInformation("Circuit half-open"));
```

---

### Best Practices

1. **Tune Thresholds**: Set `handledEventsAllowedBeforeBreaking` and `durationOfBreak` based on service reliability and recovery time.
2. **Graceful Degradation**: Return fallback responses (e.g., "Try again later") when the circuit is open.
3. **Monitoring**: Log state changes and integrate with tools like Application Insights.
4. **Test Failures**: Simulate outages to ensure the breaker behaves as expected.
5. **Combine Patterns**: Use with retry or fallback for a complete resilience strategy.

---

### Real-World Benefits
In the e-commerce API:
- **Resilience**: Prevents the API from overwhelming a failing payment service, avoiding cascading failures.
- **User Experience**: Informs users of temporary issues instead of hanging or crashing.
- **Scalability**: Reduces load on the system during outages, preserving resources for other operations.

The Circuit Breaker Pattern in .NET 8 with Polly is a powerful tool for building robust, fault-tolerant Web APIs. Let me know if you’d like to explore a specific integration (e.g., with Redis) or scenario further!
