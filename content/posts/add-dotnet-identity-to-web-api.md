---
title: "(Part 1) Add .NET Identity login and registration to a .NET 6 Web API"
date: 2022-08-17T08:46:10+01:00
draft: false
summary: "Manage the login and registration of your users on a .NET Web API using .NET Identity"
---

### GitHub Repository

All the code in this series can be found at https://github.com/dombarter/Solar.API

# The Motivation

There are a lot of tutorials out there for how to scaffold .NET Identity into your project, but that comes along with full register/login Razor pages. The idea of this post is to take the core .NET Identity logic to setup the database tables and provide key wrappers - but manage login and registering using our own API endpoints.

# NuGet Packages

You need to start by adding the following NuGet packages to your project:

- Microsoft.AspNetCore.Identity.EntityFrameworkCore
- Microsoft.EntityFrameworkCore
- Microsoft.EntityFrameworkCore.Design
- Microsoft.EntityFrameworkCore.SqlServer
- Microsoft.EntityFrameworkCore.Tools

# Database Connection

You now need to get the connection to your SQL server. Add it into your `appsettings.json` file like so:

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=localhost\\SQLEXPRESS...."
  }
}
```

# DbContext Class

Instead of making a standard `DbContext` class we need to extend the `IdentityDbContext` class which will provide us with a number of extra `DbSet`s including user and role information. Here is a basic example:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace Solar.Data
{
    public class SolarDbContext : IdentityDbContext<IdentityUser>
    {
        public SolarDbContext(DbContextOptions options) : base (options)
        {
        }
    }
}
```

# Configure Entity Framework & Identity

Now we need to configure Entity Framework, so it can connect the database and the `DbContext`, as well as configure Identity with key options such as password requirements.

Add the following lines to your startup class, `Program.cs`:

```csharp
// Add the database connection
builder.Services.AddDbContext<SolarDbContext>(options => options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// Setup identity
builder.Services.AddIdentity<IdentityUser, IdentityRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
}).AddEntityFrameworkStores<SolarDbContext>();
```

# Database Migration

With everything setup, we can now add an initial migration and apply this migration to our database:

```
add-migration init
update-database
```

**You will now notice 5 or 6 tables have been created to store all the user and role information etc.**

# Using .NET Identity

We can now login and register using the `UserManager` and `SignInManager` like below:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Solar.DTOs.Inbound;

namespace Solar.API.Controllers
{
    [ApiController]
    [Route("user")]
    public class AccountController : Controller
    {
        private readonly UserManager<IdentityUser> _userManager;
        private readonly SignInManager<IdentityUser> _signInManager;

        public AccountController(UserManager<IdentityUser> userManager, SignInManager<IdentityUser> signInManager)
        {
            _userManager = userManager;
            _signInManager = signInManager;
        }

        [HttpPost]
        [Route("register")]
        public async Task<IActionResult> Register([FromBody] RegisterDto model)
        {
            var user = new IdentityUser
            {
                UserName = model.Email,
                Email = model.Email
            };

            // Create the user
            var result = await _userManager.CreateAsync(user, model.Password);

            if (result.Succeeded)
            {
                return Ok();
            }

            return new BadRequestObjectResult(result.Errors);
        }

        [HttpPost]
        [Route("login")]
        public async Task<IActionResult> Login([FromBody] LoginDto model)
        {
            // Login
            var result = await _signInManager.PasswordSignInAsync(model.Email, model.Password, model.RememberMe, false);

            if (result.Succeeded)
            {
                return Ok();
            }

            return BadRequest("Incorrect email or password");
        }
    }
}
```

And that's it - you can now create users, and login, all using .NET Identity and managing all the DTOs and endpoint logic yourself!

# References

- https://thecodeblogger.com/2020/01/23/adding-asp-net-core-identity-to-web-api-project/
- https://www.freecodespot.com/blog/asp-net-core-identity/
