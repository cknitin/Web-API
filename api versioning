# API Versioning


## Common Versioning Strategies

There are several widely adopted strategies for API versioning. Each has its pros and cons, depending on your use case:
1. URI Path Versioning (Most Common)

    Description: The version is included in the URL path (e.g., /api/v1/resource).
    Pros: Simple, intuitive, highly discoverable via URLs.
    Cons: Can clutter the URL structure over time; requires routing adjustments.
    Example: GET /api/v1/users vs. GET /api/v2/users.

2. Query String Versioning

    Description: The version is passed as a query parameter (e.g., /api/users?api-version=1.0).
    Pros: Clean URLs, flexible for optional versioning.
    Cons: Less discoverable, might conflict with other query parameters.
    Example: GET /api/users?api-version=1.0.

3. Header Versioning

    Description: The version is specified in a custom HTTP header (e.g., Api-Version: 1.0).
    Pros: Keeps URLs clean, aligns with REST principles.
    Cons: Less visible to clients, requires client-side support for headers.
    Example: GET /api/users with header Api-Version: 1.0.

4. Media Type Versioning (Content Negotiation)

    Description: The version is embedded in the Accept header’s media type (e.g., Accept: application/vnd.myapp.v1+json).
    Pros: RESTful, leverages HTTP standards, supports content negotiation.
    Cons: Complex for clients to implement, less intuitive.
    Example: GET /api/users with Accept: application/vnd.myapp.v1+json.

```
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class TestController : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Version 1");
}

[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class TestControllerV2 : ControllerBase
{
    [HttpGet]
    public IActionResult Get() => Ok("Version 2");
}
```
