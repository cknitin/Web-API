# Refresh Token

Here’s a detailed explanation of a **real-world example** for implementing **refresh tokens** in a .NET Core Web API (using .NET 8 as of February 27, 2025). I’ll demonstrate how refresh tokens enhance security and user experience in an e-commerce scenario, with a complete implementation.

---

### What is a Refresh Token?

A **refresh token** is a long-lived credential used to obtain a new short-lived access token (e.g., JWT) when the original expires. Unlike access tokens, refresh tokens are typically opaque (not self-contained) and stored securely on the server, reducing the risk of exposure.

- **Access Token**: Short-lived (e.g., 15-30 minutes), used for API requests.
- **Refresh Token**: Long-lived (e.g., days or weeks), used to "refresh" the access token without re-authentication.

#### Why Use Refresh Tokens?
- **Security**: Short-lived access tokens limit damage if stolen; refresh tokens can be revoked.
- **User Experience**: Avoids frequent logins, keeping users authenticated seamlessly.

---

### Real-World Scenario: E-Commerce API

Imagine an e-commerce platform where:
- Users log in to browse products, place orders, and view order history.
- The API uses JWTs for authentication, but tokens expire after 15 minutes for security.
- Without refresh tokens, users would need to re-enter credentials every 15 minutes—unacceptable for a shopping app.
- With refresh tokens, users stay authenticated for days (e.g., 7 days) without re-logging in, while access tokens remain short-lived.

#### Flow
1. User logs in → Receives a short-lived JWT (access token) and a refresh token.
2. Client uses the JWT for API calls until it expires.
3. When the JWT expires (401 response), the client sends the refresh token to get a new JWT.
4. User continues shopping without interruption until the refresh token expires or is revoked.

---

### Implementation in .NET 8 Web API

#### Step 1: Setup and Packages
```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.EntityFrameworkCore.InMemory // For simplicity (use a real DB in production)
```

#### Step 2: Configure Authentication and Database
In `Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddDbContext<UserDbContext>(options => options.UseInMemoryDatabase("Users"));

// Configure JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "https://myecommerceapi.com",
            ValidAudience = "https://myecommerceapi.com",
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("your-32-character-secret-key-here!!"))
        };
    });

var app = builder.Build();

app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();

// Simple DB context for storing refresh tokens
public class UserDbContext : DbContext
{
    public DbSet<RefreshToken> RefreshTokens { get; set; }

    public UserDbContext(DbContextOptions<UserDbContext> options) : base(options) { }
}

public class RefreshToken
{
    public int Id { get; set; }
    public string Token { get; set; }
    public string UserId { get; set; }
    public DateTime Expires { get; set; }
    public bool IsRevoked { get; set; }
}
```

- **Explanation**:
  - `UserDbContext`: Stores refresh tokens (in-memory for this example; use SQL Server, Redis, etc., in production).
  - `JwtBearer`: Validates access tokens.

#### Step 3: Auth Controller with Refresh Token Logic

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
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
        private readonly UserDbContext _dbContext;
        private readonly string _issuer = "https://myecommerceapi.com";
        private readonly string _audience = "https://myecommerceapi.com";
        private readonly string _key = "your-32-character-secret-key-here!!";

        public AuthController(UserDbContext dbContext)
        {
            _dbContext = dbContext;
        }

        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginModel model)
        {
            // Simulate user validation (replace with real logic)
            if (model.Username != "customer" || model.Password != "pass123")
                return Unauthorized();

            var userId = "customer123"; // Simulated user ID
            var accessToken = GenerateAccessToken(userId);
            var refreshToken = GenerateRefreshToken();

            // Store refresh token
            await _dbContext.RefreshTokens.AddAsync(new RefreshToken
            {
                Token = refreshToken,
                UserId = userId,
                Expires = DateTime.UtcNow.AddDays(7), // 7-day lifespan
                IsRevoked = false
            });
            await _dbContext.SaveChangesAsync();

            return Ok(new { AccessToken = accessToken, RefreshToken = refreshToken });
        }

        [HttpPost("refresh")]
        public async Task<IActionResult> Refresh([FromBody] RefreshModel model)
        {
            var token = await _dbContext.RefreshTokens
                .FirstOrDefaultAsync(t => t.Token == model.RefreshToken && !t.IsRevoked);

            if (token == null || token.Expires < DateTime.UtcNow)
                return Unauthorized("Invalid or expired refresh token");

            var newAccessToken = GenerateAccessToken(token.UserId);
            var newRefreshToken = GenerateRefreshToken();

            // Revoke old refresh token and store new one
            token.IsRevoked = true;
            await _dbContext.RefreshTokens.AddAsync(new RefreshToken
            {
                Token = newRefreshToken,
                UserId = token.UserId,
                Expires = DateTime.UtcNow.AddDays(7),
                IsRevoked = false
            });
            await _dbContext.SaveChangesAsync();

            return Ok(new { AccessToken = newAccessToken, RefreshToken = newRefreshToken });
        }

        private string GenerateAccessToken(string userId)
        {
            var claims = new[]
            {
                new Claim(ClaimTypes.NameIdentifier, userId),
                new Claim(ClaimTypes.Role, "Customer")
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_key));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer: _issuer,
                audience: _audience,
                claims: claims,
                expires: DateTime.UtcNow.AddMinutes(15), // Short-lived
                signingCredentials: creds);

            return new JwtSecurityTokenHandler().WriteToken(token);
        }

        private string GenerateRefreshToken()
        {
            return Guid.NewGuid().ToString(); // Simple random token (use crypto-secure in production)
        }
    }

    public class LoginModel
    {
        public string Username { get; set; }
        public string Password { get; set; }
    }

    public class RefreshModel
    {
        public string RefreshToken { get; set; }
    }
}
```

- **Explanation**:
  - **Login**: Validates credentials, issues a 15-minute JWT and a 7-day refresh token, storing the latter in the DB.
  - **Refresh**: Validates the refresh token, revokes it, issues a new JWT and refresh token, and stores the new refresh token.
  - **Access Token**: Short-lived for security.
  - **Refresh Token**: Opaque, tied to a user, and revocable.

#### Step 4: Secure an Endpoint
```csharp
[ApiController]
[Route("[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [Authorize(Roles = "Customer")]
    public IActionResult GetOrders()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return Ok(new { UserId = userId, Orders = new[] { "Order1", "Order2" } });
    }
}
```

---

### Real-World Flow

#### 1. User Logs In
- **Request**: `POST /auth/login` with `{"username": "customer", "password": "pass123"}`
- **Response**:
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  }
  ```

#### 2. User Accesses Orders
- **Request**: `GET /orders` with `Authorization: Bearer <accessToken>`
- **Response**: `{"userId": "customer123", "orders": ["Order1", "Order2"]}`

#### 3. Access Token Expires (After 15 Minutes)
- **Request**: `GET /orders` → `401 Unauthorized`

#### 4. Client Requests New Token
- **Request**: `POST /auth/refresh` with `{"refreshToken": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"}`
- **Response**:
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...new...",
    "refreshToken": "new-refresh-token-1234-5678-9012-3456"
  }
  ```

#### 5. User Resumes Shopping
- Uses the new access token for subsequent requests until it expires again.

#### 6. Refresh Token Expires or Revoked
- After 7 days (or if revoked, e.g., user logs out), `POST /auth/refresh` returns `401`, forcing re-login.

---

### Advanced Considerations

1. **Secure Storage**:
   - Store refresh tokens in a secure database (e.g., SQL Server with hashed values) or Redis for distributed systems.
   - Use a cryptographically secure random generator (e.g., `RandomNumberGenerator`) instead of `Guid`:
     ```csharp
     using System.Security.Cryptography;
     private string GenerateRefreshToken()
     {
         var bytes = RandomNumberGenerator.GetBytes(32);
         return Convert.ToBase64String(bytes);
     }
     ```

2. **Revocation**:
   - Add a logout endpoint to revoke refresh tokens:
     ```csharp
     [HttpPost("logout")]
     [Authorize]
     public async Task<IActionResult> Logout()
     {
         var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
         var tokens = await _dbContext.RefreshTokens.Where(t => t.UserId == userId && !t.IsRevoked).ToListAsync();
         tokens.ForEach(t => t.IsRevoked = true);
         await _dbContext.SaveChangesAsync();
         return Ok();
     }
     ```

3. **Token Rotation**:
   - Issuing a new refresh token on each refresh (as shown) enhances security by limiting reuse.

4. **HTTPS**: Enforce HTTPS to protect tokens in transit.

---

### Best Practices

- **Short Access Tokens**: Keep JWTs short-lived (15-60 minutes).
- **Long Refresh Tokens**: Set a reasonable lifespan (e.g., 7-30 days) based on UX/security needs.
- **Revocation**: Track and revoke refresh tokens on logout or suspicious activity.
- **Secure Keys**: Store the JWT signing key in a secret manager (e.g., Azure Key Vault).
- **Monitoring**: Log refresh token usage for audit trails.

---

### Real-World Benefits
In this e-commerce example:
- **Security**: A stolen JWT is only valid for 15 minutes, minimizing risk.
- **User Experience**: Customers shop for days without re-authenticating.
- **Scalability**: Stateless JWTs scale easily, with refresh tokens managed server-side.

This implementation in .NET 8 provides a robust, secure, and user-friendly authentication flow. Let me know if you’d like to explore a specific aspect (e.g., revocation, OAuth integration) further!
