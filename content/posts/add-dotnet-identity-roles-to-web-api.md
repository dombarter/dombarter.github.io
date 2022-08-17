---
title: "(Part 2) Add .NET Identity roles to a .NET 6 Web API"
date: 2022-08-17T09:08:06+01:00
draft: false
summary: "Manage your user's roles on a .NET Web API using .NET Identity"
---

### GitHub Repository

All the code in this series can be found at https://github.com/dombarter/Solar.API

# Motivation

In my [previous post](/posts/add-dotnet-identity-to-web-api), I explored how to add .NET Identity to a .NET 6 Web API project. We managed to register users and then login as them. Now it would be good to add roles to those users - so once we have a token/session system setup we can restrict users to different endpoints.

# Amend The Startup

The first thing we need to do is tell Identity we are going to be using roles. Go to your startup class, `Program.cs`:

```csharp
builder.Services.AddIdentity<IdentityUser, IdentityRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
})
.AddRoles<IdentityRole>() // <--- add this line
.AddEntityFrameworkStores<SolarDbContext>();
```

# Define The Roles

Next, it is a good idea to create a class where we define the possible roles in the system rather than relying on 'magic strings'. Create this class, `Roles.cs`, somewhere:

```csharp
namespace Solar.Common.Roles
{
    public static class Roles
    {
        public const string Admin = "Admin";
        public const string User = "User";
    }
}
```

# Store The Roles

Now we need to make sure these roles are added to the database, so we can link them to our current user. The best place to to do this is in our startup class, `Program.cs`. I've also added a line that will make sure our database os migrated each time we startup - this way we will never miss out on a change:

```csharp
...
var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;

    // Migrate the database
    var db = services.GetRequiredService<SolarDbContext>();
    db.Database.Migrate();

    // Add the roles
    var roleManager = services.GetRequiredService<RoleManager<IdentityRole>>();
    if (!await roleManager.RoleExistsAsync(Roles.Admin))
    {
        await roleManager.CreateAsync(new IdentityRole(Roles.Admin));
    }
    if (!await roleManager.RoleExistsAsync(Roles.User))
    {
        await roleManager.CreateAsync(new IdentityRole(Roles.User));
    }
}
```

# Assign Roles

We can now assign roles to a user like so (register action):

```csharp
[HttpPost]
[Route("register")]
[AllowAnonymous]
public async Task<IActionResult> Register([FromBody] RegisterDto model)
{
    var user = new IdentityUser
    {
        UserName = model.Email,
        Email = model.Email
    };

    var createResult = await _userManager.CreateAsync(user, model.Password);

    if (!createResult.Succeeded)
    {
        return new BadRequestObjectResult(createResult.Errors);
    }

    // Assign User role
    var assignRoleResult = await _userManager.AddToRoleAsync(user, Roles.User);

    if (!assignRoleResult.Succeeded)
    {
        return new BadRequestObjectResult(assignRoleResult.Errors);
    }

    return Ok();
}
```

# Authorization Attributes

Currently roles will have no effect until we setup a session/token system (watch out for my next post!), however once this is setup, we will be able to apply the `Authorize` attributes at the controller and action level like so:

```csharp
[ApiController]
[Route("moons")]
[Authorize]
public class MoonController : Controller
{
    private readonly List<string> Moons = new List<string> { "Moon", "Europa", "Titan", "Ganymede", "Milmas", "Hyperion", "Dione", "Kiviuq" };
    private readonly Random Random = new Random();

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
}
```

# References

- https://docs.microsoft.com/en-us/aspnet/core/security/authorization/roles?view=aspnetcore-6.0
- https://www.c-sharpcorner.com/article/jwt-authentication-and-authorization-in-net-6-0-with-identity-framework/
- https://docs.microsoft.com/en-us/aspnet/core/security/authorization/secure-data?view=aspnetcore-6.0
