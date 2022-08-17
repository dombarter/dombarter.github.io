---
title: "(Part 3) Add JWT based authentication and authorization to a .NET 6 Web API"
date: 2022-08-17T09:28:40+01:00
draft: false
summary: "Authenticate and authorize users calling your .NET 6 Web API by using JSON Web Tokens (JWTs)"
---

### GitHub Repository

All the code in this series can be found at https://github.com/dombarter/Solar.API

# Motivation

In my [previous post](/posts/add-dotnet-jwts-to-web-api), I explored how to manage login, registration and role assignment of our users using .NET Identity. In this post I am going to add JWT based authentication and authorization to our .NET 6 Web API.

# What are JWTs?

> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed.

(https://jwt.io/introduction)

JWTs are becoming increasingly popular in the world of single sign on, and are a very scalable, stateless and memory efficient way of providing authentication.

# NuGet Packages

We need to install the following package:

- Microsoft.AspNetCore.Authentication.JwtBearer

# Define The JWT Configuration

JWTs are defined by some key parts including:

- Issuer (who has issued the key)
- Audience (who the key is intended for)
- Signing Key (a private random string used to sign the key)

We need to define these in our `appsettings.json`:

```json
{
  "Jwt": {
    "Key": "ThisIsMySecretKey",
    "Issuer": "localhost",
    "Audience": "localhost"
  }
}
```

Don't worry too much about the value of the `Issuer` and `Audience`, just make sure they're the same. When used in a single sign on setting - we might have one machine generate the key, which could then be consumed by multiple different places. In our case we're creating and consuming it in the same place. It's just a straight string comparison.

For more information see https://www.rfc-editor.org/rfc/rfc7519#section-4.1.

# Amend The Startup

Now we need to alter our startup class, `Program.cs` to configure our authentication mechanism. Specifically we are telling the Web API that it should look out for a JWT in the request headers, and use this to authenticate the current user.

```csharp
// Add JWTs
builder.Services.AddAuthentication(auth =>
{
    auth.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    auth.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(options =>
{
    options.SaveToken = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
    };
});

...

// Must be in this order
app.UseAuthentication();
app.UseAuthorization();
```

# Token Service

We now need to make a token service, that will accept an `IdentityUser` and an expiry time, before generating a JWT we can return to the user.

```csharp
// TokenService.cs

using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;

namespace Solar.Services.Token
{
    public class TokenService : ITokenService
    {
        private readonly UserManager<IdentityUser> _userManager;
        private readonly IConfiguration _config;

        public TokenService(UserManager<IdentityUser> userManager, IConfiguration config)
        {
            _userManager = userManager;
            _config = config;
        }

        async public Task<string> GenerateJwtToken(IdentityUser user, TimeSpan expiration)
        {
            // Define the token claims (username and unique guid)
            var claims = new List<Claim>
            {
                new Claim(JwtRegisteredClaimNames.Sub, user.UserName),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            };

            // Add the roles to the token
            foreach(var role in await _userManager.GetRolesAsync(user))
            {
                claims.Add(new Claim("role", role));
            }

            // Encode our private JWT key
            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            // Put everything together
            var token = new JwtSecurityToken(
                issuer: _config["Jwt:Issuer"],
                audience: _config["Jwt:Audience"],
                expires: DateTime.UtcNow.Add(expiration),
                claims: claims,
                signingCredentials: creds
            );

            // Build the token as a string
            return new JwtSecurityTokenHandler().WriteToken(token);
        }
    }
}
```

# Consuming The Token Service

Now we have the token service setup we need to register it in the dependency injection container within `Program.cs`:

```csharp
// Add token service
builder.Services.AddTransient<ITokenService, TokenService>();
```

We can then utilise the token service in our login action like so:

```csharp
[HttpPost]
[Route("login")]
[AllowAnonymous]
public async Task<ActionResult<string>> Login([FromBody] LoginDto model)
{
    var result = await _signInManager.PasswordSignInAsync(model.Email, model.Password, model.RememberMe, false);

    if (!result.Succeeded)
    {
        return BadRequest("Incorrect email or password");
    }

    // Generate JWT
    var user = await _userManager.FindByNameAsync(model.Email);
    var token = await _tokenService.GenerateJwtToken(user, TimeSpan.FromMinutes(30));

    return Ok(token);
}
```

# Role Based Authorization

Because we have configured our JWTs to include the role information, this means we can safely use the `Authorize` attributes across our controllers and actions:

```csharp
[HttpGet]
[Route("one")]
[Authorize(Roles = "User")]
public ActionResult<string> GetRandomMoon()
{
    return Moons[Random.Next(Moons.Count)];
}

[HttpGet]
[Route("two")]
[Authorize(Roles = "Admin")]
public ActionResult<string> GetTwoRandomMoons()
{
    return $"{Moons[Random.Next(Moons.Count)]}, {Moons[Random.Next(Moons.Count)]}";
}
```

# Accessing The Current User Information

Because we configured our JWTs to include the current username, this means we can grab the information of the user who the token belongs to - which could be helpful when making SQL etc.

```csharp
[HttpGet]
[Route("user")]
[Authorize(Roles = "Admin, User")]
public async Task<ActionResult<IdentityUser>> GetLoggedInUser()
{
    var username = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var user = await _userManager.FindByNameAsync(username);
    return new OkObjectResult(user);
}
```

# Supporting JWT support to Swagger

And finally, if you'd like to support the Authorize window in Swagger (adds the ability to pass the Bearer token with each subsequent request), add the following to your startup class:

```csharp
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    var jwtSecurityScheme = new OpenApiSecurityScheme
    {
        Name = "Authorization",
        Type = SecuritySchemeType.Http,
        Scheme = JwtBearerDefaults.AuthenticationScheme,
        BearerFormat = "JWT",
        In = ParameterLocation.Header,
        Reference = new OpenApiReference
        {
            Type = ReferenceType.SecurityScheme,
            Id = JwtBearerDefaults.AuthenticationScheme
        }
    };

    options.AddSecurityDefinition(JwtBearerDefaults.AuthenticationScheme, jwtSecurityScheme);
    options.AddSecurityRequirement(new OpenApiSecurityRequirement(){{ jwtSecurityScheme, new string[] {} }});
});
```

# Testing The JWTs Using Swagger

Load up swagger, login using your username and password, and grab the JWT from the response:

![login](/images/jwt-header-swagger/login.png)

Now scroll to the top of the page, and click on the `Authorize` button and paste in your token:

![authorize](/images/jwt-header-swagger/authorize.png)

Make sure to press `Authorize` to save the token!

And finally, scroll to one of the endpoints that requires authentication and test it out. If you have a valid token and the correct roles - you will see the content returned. If not you'll get a contextual http response (401 etc):

![moons](/images/jwt-header-swagger/moons.png)

# References

- https://docs.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-6.0
- https://codewithmukesh.com/blog/aspnet-core-api-with-jwt-authentication/
- https://weblog.west-wind.com/posts/2021/Mar/09/Role-based-JWT-Tokens-in-ASPNET-Core
- https://www.c-sharpcorner.com/article/how-to-add-jwt-bearer-token-authorization-functionality-in-swagger/
- https://www.freecodespot.com/blog/use-jwt-bearer-authorization-in-swagger/
