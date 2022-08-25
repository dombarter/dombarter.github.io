---
title: "Add integration tests to an existing .NET 6 Web API using Entity Framework, Identity and JWTs"
date: 2022-08-25T15:08:39+01:00
draft: false
summary: "Test the full lifecycle of your API controllers using integration tests. Combined with in-memory databases you can fully mock your system and reproducible results. This approach even deals with cookies, so you can authenticate using a JWT in a cookie."
---

### GitHub Repository

A full example of this code can be found at https://github.com/dombarter/Solar.API

# Motivation

As you may have seen from my previous posts, I've been experimenting with creating a .NET 6 Web API that uses Entity Framework, Identity and JWTs as cookies that can be consumed by a Vue.js SPA. I've not delved into the world of integration tests too much but thought it would be a really interesting concept to create an entire test database and test our full controller lifecycle, as this will catch errors at any level.

# Create Your Test Project

The first step is to make a `xUnit` test project to complement your `API` project.

Our `API` project is called `Solar.API` and we created a new `xUnit` project called `Solar.API.Test`.

# NuGet Packages

You need to install the following NuGet packages:

- `Microsoft.AspNetCore.Mvc.Testing` - for mocking the controllers
- `Microsoft.EntityFrameworkCore.InMemory` - for creating an empty test database

# API Startup Class

Our integration tests need to access our startup class, `Program.cs` so they can create a mock of the controllers. We need to make some slight adjustments to the access levels because by default it is only internally accessible.

Go to `Program.cs` and add this line at the bottom of the file:

```csharp
app.Run();

public partial class Program {} // <-- add this line!
```

# Integration Tests Base Class

Now we need to make a base class that all our integration tests will inherit. In this class we will provide a few important things:

- `GenerateClient` - for creating a mock of the controllers, and swapping out our SQL Server for an empty in-memory equivalent
- Authentication helper methods

## GenerateClient

The `GenerateClient` performs the following:

- Creates a new `WebApplicationFactory` based on our `API` startup class
- It then amends the startup class, by changing some of the services
- It firstly removes the existing `DbContext` service
- It then replaces it with a new `DbContext` class that uses an in-memory database with a unique name (this ensures each integration test has an empty database to work with)
- It finally defines a custom `https` endpoint for the mocked controller, meaning our `Secure` JWT cookies can be transmitted

```csharp
public class IntegrationTestsBase
{
    protected HttpClient GenerateClient()
    {
        // Build up the API, using an in memory database
        var application = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Remove the db context
                    services.RemoveAll(typeof(DbContextOptions<SolarDbContext>));

                    // Replace the connection with an in memory equivalent
                    var dbName = Guid.NewGuid().ToString();
                    services.AddDbContext<SolarDbContext>(options =>
                    {
                        options.UseInMemoryDatabase(dbName);
                    });
                });
            });

        // Generate the client
        var client = application.CreateClient(new WebApplicationFactoryClientOptions { BaseAddress = new Uri("https://localhost") });
        return client;
    }
}
```

# Write Some Tests!

You can now write some tests. Here are some examples of what you can do:

## Making Sure Endpoints Are Protected

in this example we make `GET` requests to multiple endpoints we know should be protected under normal circumstances. We make sure that a 401 code is returned.

```csharp
public class MoonControllerTests : IntegrationTestsBase
{
    [Theory]
    [InlineData("/moons/one")]
    [InlineData("/moons/two")]
    public async Task Get_WhenUnauthenticated_ReturnsUnauthorized(string url)
    {
        // Arrange
        var client = GenerateClient();

        // Act
        var response = await client.GetAsync(url);

        // Assert
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
}
```

## Registering & Logging In

In this example we register a new user, then login as them and make sure that both requests return an `OK` response indicating the request was a success.

```csharp
public class AccountControllerTests : IntegrationTestsBase
{
    [Fact]
    public async Task Post_Login_ReturnsOK()
    {
        // Arrange
        var client = GenerateClient();

        // Act
        var register = await Register(client, new RegisterDto
        {
            Email = "dom@email.com",
            Password = "password"
        });

        var login = await Login(client, new LoginDto
        {
            Email = "dom@email.com",
            Password = "password"
        });

        // Assert
        Assert.Equal(HttpStatusCode.OK, register.StatusCode);
        Assert.Equal(HttpStatusCode.OK, login.StatusCode);
    }
}
```

Where `Login` and `Register` are some helper methods I was talking about earlier:

```csharp
protected async Task<HttpResponseMessage> Register(HttpClient client, RegisterDto registerDto)
{
    // Act
    var response = await client.PostAsJsonAsync("/user/register", registerDto);
    return response;
}

protected async Task<HttpResponseMessage> Login(HttpClient client, LoginDto loginDto)
{
    // Act
    var response = await client.PostAsJsonAsync("/user/login", loginDto);
    return response;
}
```

## Requesting An Endpoint When Authenticated

In this example, we authenticate the request by registering and logging in using a helper method. This stores a JWT cookie. We then make a request to an endpoint that would usually be protected and make sure that the response is `OK`.

```csharp
public class MoonControllerTests : IntegrationTestsBase
{
    [Fact]
    public async Task Get_OneMoon_AsUser()
    {
        // Arrange
        var client = GenerateClient();
        await AuthenticateAsUser(client);

        // Act
        var response = await client.GetAsync("/moons/one");

        // Assert
        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

# References

Based on the following websites:

- https://docs.microsoft.com/en-us/aspnet/core/test/integration-tests?view=aspnetcore-6.0
- https://stackoverflow.com/questions/70900451/c-sharp-netcore-webapi-integration-testing-httpclient-uses-https-for-get-reques
