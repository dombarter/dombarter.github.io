---
title: "Validate JWT cookies sent to an Azure Functions application"
date: 2022-08-22T09:20:04+01:00
draft: false
summary: "Validate JWTs as cookies, that are sent to an Azure Functions application"
---

# Motivation

I have a Vue.js SPA that communicates with a .NET 6 Web API for the majority of things. It uses JWTs in cookies for authentication and authorisation. The API generates the JWTs. However, I would like some Azure Functions (microservices) to perform scalable operations as to not stress the API. These Azure Functions will need authenticating otherwise anyone can call them.

Let's authenticate them using the JWT cookie our API has given us.

# NuGet Packages

Install the following NuGet package.

- System.IdentityModel.Tokens.Jwt (<= 6.10.2)

The package version must be lower than 6.11 due to https://github.com/Azure/azure-functions-host/issues/7878.

# App Settings

We need to set some details about our JWTs in our `local.settings.json` so they can be validated. These should match the details used by the API where the JWT is generated.

```json
{
  "Values": {
    "JwtKey": "ThisIsMySecretKey",
    "JwtIssuer": "https://localhost:7234/",
    "JwtAudience": "https://localhost:7234/",
    "JwtCookieName": "solar-access-token"
  }
}
```

# Token Validator

Now we need to make a method (could be static, or passed via dependency injection), that will validate the request:

```csharp
public static bool TokenIsValid(HttpRequest req)
{
    try
    {
        string cookie = req.Cookies[Environment.GetEnvironmentVariable("JwtCookieName")] ?? "";

        var tokenParams = new TokenValidationParameters()
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = Environment.GetEnvironmentVariable("JwtIssuer"),
            ValidAudience = Environment.GetEnvironmentVariable("JwtAudience"),
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Environment.GetEnvironmentVariable("JwtKey")))
        };

        new JwtSecurityTokenHandler().ValidateToken(cookie, tokenParams, out var securityToken);
        return true;
    }
    catch
    {
        return false;
    }
}
```

# Token Validation

We can now call this method in each Function, and the JWT within the cookie will be validated:

```csharp
public static async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req)
{
    // Validate the access token
    if (!TokenValidator.TokenIsValid(req))
    {
        return new UnauthorizedResult();
    }
}
```

# References

- https://docs.microsoft.com/en-us/dotnet/api/system.identitymodel.tokens.jwt.jwtsecuritytokenhandler.validatetoken?view=azure-dotnet
