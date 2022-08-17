---
title: "(Part 4) Amend JWTs to work securely with SPAs in a .NET 6 Web API"
date: 2022-08-17T13:13:12+01:00
draft: false
summary: "Change how we handle JWTs so that they can be securely used by SPAs rather than the standard API lifecycle"
---

### GitHub Repository

All the code in this series can be found at https://github.com/dombarter/Solar.API

# Motivation

In my [previous post](/posts/add-dotnet-jwts-to-web-api), we setup a full .NET 6 Web API with user and role management with .NET Identity, and then we've authorised and authenticated those users using JWTs.

However, there are security risks surrounding returning the JWT as plain text and letting the client handle the storage of it. This is because the most likely place it would be stored is in `localStorage` which is highly susceptible to XSS (cross-site scripting). If you were consuming this application using Postman or a mobile application this wouldn't be a problem. But for an SPA (single page application) it is.

Instead, we are going to change the way the JWT is returned, so that it is returned as an expirable, http only cookie; which cannot be accessed by javascript and will be automatically included on every request.

# Cookie Name

We first of all need to define a name for our access token cookie so we're always reading and writing to the same one. This needs to be added in `appsettings.json` under the `Jwt` section:

```json
{
  "Jwt": {
    "Key": "ThisIsMySecretKey",
    "Issuer": "https://localhost:7234/",
    "Audience": "https://localhost:7234/",
    "CookieName": "solar-access-token" // <- add this line
  }
}
```

# Alter The Authentication Mechanism

Now we need to go to the startup class, `Program.cs` and add an extra option to the `AddJwtBearer` method which will pull the token from a cookie rather than from its default location in the `Authorization` header:

```csharp
options.Events = new JwtBearerEvents
{
    OnMessageReceived = context =>
    {
        context.Token = context.Request.Cookies[builder.Configuration["Jwt:CookieName"]];
        return Task.CompletedTask;
    },
};
```

# Generate The Cookie On Login

We're now reading the JWT from the cookie, but we need to generate the cookie and send it to the client as well. Amend your login action so it appends a cookie to the response and only returns account information in the body (no access tokens!):

```csharp
Response.Cookies.Append(_config["Jwt:CookieName"], token, new CookieOptions
{
    HttpOnly = true,
    IsEssential = true,
    MaxAge = TimeSpan.FromMinutes(30),
    SameSite = SameSiteMode.None,
    Secure = true,
});

return new OkObjectResult(
    new LoginResultDto(user.Email, await _userManager.GetRolesAsync(user))
);
```

# Clear The Cookie On Logout

Because the client cannot access the cookie at all, the only way we can clear it is by calling a logout action, that will return an empty cookie with an instant expiry:

```csharp
[HttpPost]
[Route("logout")]
[AllowAnonymous]
public IActionResult Logout()
{
    Response.Cookies.Append(_config["Jwt:CookieName"], string.Empty, new CookieOptions
    {
        HttpOnly = true,
        IsEssential = true,
        MaxAge = TimeSpan.Zero,
        SameSite = SameSiteMode.None,
        Secure = true,
    });

    return Ok();
}
```

# Update The Swagger Configuration

Now that we no longer need to manually include the JWT with each request, we can remove the Swagger configuration that gave us that `Authorization` button at the top right:

```csharp
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

becomes

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
```

# Testing

You should now be able to open the Swagger page for your API, make a login request and then make a call to a protected endpoint and you will get a successful response. If you look in the dev tools you should notice the `solar-access-token` has been set. Try removing it and see what happens!

# Connecting To A Vue.js SPA

Cookies work when using a system such as Swagger, but when using them in Vue.js we need a bit more configuration. This is because it is another web site, so CORs comes into play.

First of all we need to add a default CORs policy to our API. Make sure to define specific origins rather than allowing all. This is applied in `Program.cs`:

```csharp
builder.Services.AddCors(options => options.AddDefaultPolicy(
    policy => policy
        .WithOrigins("http://localhost:8080")
        .AllowCredentials()
        .AllowAnyHeader()
        .AllowAnyMethod()
));

...

app.UseCors();
```

And then finally, if you are using a http client such as `axios` - make sure to set `withCredentials` to `true` to make sure those cookies are always sent and received:

```ts
const axiosInstance = axios.create({
  withCredentials: true,
});
```

# Conclusion

And that's it, we now have a way of securely managing users, roles and access to our API when calling from a JavaScript SPA.

# References

- https://javascript.plainenglish.io/how-to-secure-jwt-in-a-single-page-application-6a46e69fc393
- https://spin.atomicobject.com/2020/07/25/net-core-jwt-cookie-authentication/
- https://dotnetcoretutorials.com/2017/01/15/httponly-cookies-asp-net-core/
- https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.http.cookieoptions?view=aspnetcore-6.0#properties
- https://blog.logrocket.com/jwt-authentication-best-practices/
- https://stackoverflow.com/questions/71419379/set-cookie-not-working-properly-in-axios-call
