# Authentication/Authorization

Here's a detailed explanation and implementation of **Authentication** and **Authorization** in a .NET Core Web API, focusing on the latest versions (e.g., .NET 6, 7, or 8 as of February 27, 2025). These mechanisms secure your API by verifying user identity (authentication) and controlling access to resources (authorization).

---

### What are Authentication and Authorization?

- **Authentication**: Determines "who" a user is by verifying credentials (e.g., username/password, token). It answers, "Are you who you say you are?"
- **Authorization**: Determines "what" a user can do based on their identity or roles. It answers, "Do you have permission to do this?"

---

### Authentication in .NET Core Web API

#### Common Schemes
1. **JWT (JSON Web Tokens)**: Stateless, widely used for APIs.
2. **OAuth 2.0/OpenID Connect**: For delegated authorization and identity (e.g., via Google, Azure AD).
3. **Basic Authentication**: Simple but less secure (username/password in headers).
4. **ASP.NET Core Identity**: Full-featured identity management with database-backed users.

Below, I’ll focus on **JWT Authentication**, the most common approach for Web APIs.

---

### Implementing JWT Authentication in .NET 8

#### Step 1: Install Packages
```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

#### Step 2: Configure JWT Authentication
In `Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();

// Configure JWT Authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = "https://myapi.example.com",
        ValidAudience = "https://myapi.example.com",
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("your-32-character-secret-key-here!!")) // 256-bit key
    };
});

var app = builder.Build();

// Configure the HTTP request pipeline
app.UseRouting();
app.UseAuthentication(); // Add authentication middleware
app.UseAuthorization();
app.MapControllers();

app.Run();
```

- **Explanation**:
  - **`AddAuthentication`**: Registers JWT as the default scheme.
  - **`AddJwtBearer`**: Configures JWT validation:
    - `ValidIssuer/Audience`: Ensures tokens are from/to the expected source.
    - `IssuerSigningKey`: Verifies token signature (use a secure, long key in production).
    - `ValidateLifetime`: Rejects expired tokens.

#### Step 3: Generate JWT Tokens
Add a controller to issue tokens:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

namespace MyWebApi.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly IConfiguration _config;

        public AuthController(IConfiguration config)
        {
            _config = config;
        }

        [HttpPost("login")]
        public IActionResult Login([FromBody] LoginModel model)
        {
            // Simulate user validation (replace with real logic)
            if (model.Username != "test" || model.Password != "password")
                return Unauthorized();

            var claims = new[]
            {
                new Claim(ClaimTypes.Name, model.Username),
                new Claim(ClaimTypes.Role, "User")
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("your-32-character-secret-key-here!!"));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer: "https://myapi.example.com",
                audience: "https://myapi.example.com",
                claims: claims,
                expires: DateTime.Now.AddMinutes(30),
                signingCredentials: creds);

            return Ok(new { token = new JwtSecurityTokenHandler().WriteToken(token) });
        }
    }

    public class LoginModel
    {
        public string Username { get; set; }
        public string Password { get; set; }
    }
}
```

- **Explanation**:
  - Generates a JWT with user claims (e.g., name, role).
  - Token expires in 30 minutes and is signed with the same key used for validation.

#### Step 4: Secure an Endpoint
```csharp
[ApiController]
[Route("[controller]")]
public class WeatherController : ControllerBase
{
    [HttpGet]
    [Authorize] // Requires authentication
    public IActionResult GetWeather()
    {
        var userName = User.Identity.Name; // From JWT claim
        return Ok(new { Temperature = 25, Condition = "Sunny", User = userName });
    }
}
```

- **Test**:
  - **Request**: `POST /auth/login` with `{"username": "test", "password": "password"}`
    - Response: `{"token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}`
  - **Request**: `GET /weather` with header `Authorization: Bearer <token>`
    - Response: `{"Temperature": 25, "Condition": "Sunny", "User": "test"}`
  - Without token: `401 Unauthorized`.

---

### Authorization in .NET Core Web API

#### Role-Based Authorization
Restrict access by roles:

```csharp
[HttpGet("admin")]
[Authorize(Roles = "Admin")] // Requires "Admin" role
public IActionResult GetAdminWeather()
{
    return Ok(new { Temperature = 25, Condition = "Admin Sunny" });
}
```

- **Explanation**: The JWT must include a `role` claim with value "Admin".

#### Policy-Based Authorization
Define custom policies for finer control:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AtLeast18", policy =>
        policy.RequireClaim("age", "18", "19", "20", "21", "22", "23", "24", "25")); // Example: Age >= 18
});

// Controller
[HttpGet("restricted")]
[Authorize(Policy = "AtLeast18")]
public IActionResult GetRestrictedWeather()
{
    return Ok(new { Temperature = 25, Condition = "Restricted Sunny" });
}
```

- **Explanation**: Requires an `age` claim with a value of 18 or higher.

#### Custom Authorization Handler
Implement complex logic:

```csharp
using Microsoft.AspNetCore.Authorization;

public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        var ageClaim = context.User.FindFirst("age");
        if (ageClaim != null && int.TryParse(ageClaim.Value, out var age) && age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}

// Register in Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Adult", policy => policy.Requirements.Add(new MinimumAgeRequirement(18)));
});
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();

// Controller
[HttpGet("adult")]
[Authorize(Policy = "Adult")]
public IActionResult GetAdultWeather()
{
    return Ok(new { Temperature = 25, Condition = "Adult Sunny" });
}
```

- **Explanation**: Custom handler checks the `age` claim dynamically.

---

### Advanced Features

#### 1. Refresh Tokens
Extend JWT lifetime securely:

```csharp
[HttpPost("refresh")]
public IActionResult RefreshToken([FromBody] string refreshToken)
{
    // Validate refresh token (e.g., from DB)
    if (refreshToken != "valid-refresh-token") return Unauthorized();

    var token = GenerateNewJwt(); // Same logic as in Login
    return Ok(new { token });
}

private JwtSecurityToken GenerateNewJwt() => /* Logic from Login */;
```

#### 2. OpenID Connect with Azure AD
```csharp
builder.Services.AddAuthentication()
    .AddMicrosoftIdentityWebApi(builder.Configuration.GetSection("AzureAd"));

/* appsettings.json */
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "Domain": "yourdomain.onmicrosoft.com",
    "TenantId": "your-tenant-id",
    "ClientId": "your-client-id",
    "Audience": "api://your-client-id"
  }
}
```

- **Explanation**: Integrates with Azure AD for enterprise authentication.

#### 3. Claims Transformation
Modify claims post-authentication:

```csharp
builder.Services.AddTransient<IClaimsTransformation, CustomClaimsTransformation>();

public class CustomClaimsTransformation : IClaimsTransformation
{
    public Task<ClaimsPrincipal> TransformAsync(ClaimsPrincipal principal)
    {
        var identity = (ClaimsIdentity)principal.Identity;
        identity.AddClaim(new Claim("custom", "value"));
        return Task.FromResult(principal);
    }
}
```

---

### Best Practices

1. **Secure Keys**: Store JWT keys in environment variables or a secret manager (e.g., Azure Key Vault).
2. **Short-Lived Tokens**: Use short expiration (e.g., 15-30 minutes) with refresh tokens.
3. **HTTPS Only**: Enforce HTTPS to protect tokens in transit.
4. **Minimal Claims**: Include only necessary claims to reduce token size.
5. **Logging**: Log authentication failures for security monitoring, but avoid sensitive data.

---

### Real-World Scenario
Imagine an e-commerce API:
- **Login**: `POST /auth/login` issues a JWT with `role: "Customer"`.
- **Products**: `GET /products` requires authentication (`[Authorize]`).
- **Admin Dashboard**: `GET /admin` requires `role: "Admin"` (`[Authorize(Roles = "Admin")]`).
- **Age-Restricted Items**: `GET /restricted` uses a policy (`[Authorize(Policy = "Adult")]`).

Sample JWT:
```json
{
  "sub": "test",
  "role": "Customer",
  "age": "20",
  "iss": "https://myapi.example.com",
  "aud": "https://myapi.example.com",
  "exp": 1677654321
}
```

---

Authentication and Authorization in .NET 8 Web API provide robust, flexible security options, with JWT being the go-to for stateless APIs. Let me know if you’d like to explore a specific scheme (e.g., OAuth) or scenario further!

