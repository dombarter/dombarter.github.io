---
title: "Adding app settings and user secrets to an Azure Functions project"
date: 2022-02-10T08:09:35Z
draft: false
summary: "How to separate the concerns of environment configuration and application configuration in an azure functions project and distribute sensitive settings via user secrets and environment variables."
---

# The Motivation

Currently when you make an Azure Functions project you have a `local.settings.json` file where you setup environment rules do do with the runtime frameworks, CORs settings etc - you can also configure 'application settings' that will be exposed as environment variables during runtime. 

There are two things I would like to change about this:
* I would like to separate the concerns by keeping environment configuration in one place, and my application/logic specific settings elsewhere.
* It is not best practice to distribute sensitive settings by committing them into the repository - we need a different way of doing this. Currently you would have to commit the `local.settings.json` to distribute the sensitive values. 

# Setup

## Enable User Secrets

> User secrets are a tool built into .NET to allow developers to store secret information outside of the project root. It makes it less likely that secrets are accidentally committed to source control. 

Right click on the project in Visual Studio and select `Manage User Secrets` - this will add any required NuGet packages and alter the project file where necessary. 

Once you have followed the required steps you should be able to click on `Manage User Secrets` again and an empty `secrets.json` file will open. This indicates that user secrets has been correctly setup. 

## Add `appsettings.json` file

In the root of your project create an `appsettings.json` file and setup the insensitive values you want to store. Here is an example. 

```json
// appsettings.json

{
  "ConnectionStrings": {
    "MyConnectionString": "" // This value will be stored in user secrets - hence empty
    // (but we put it in here to remind us there is actually a value somewhere!)
  },
  "General": {
    "RandomColour": "blue",
    "Shape": "triangle"
  }
}
```

You also need to make sure that the `appsettings.json` file is set to copy to your build output: 
```xml
<None Update="appsettings.json">
  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</None>
```

## Add sensitive settings to User Secrets

You can see above that we store insensitive values directly in `appsettings.json`, but we will merge together our user secrets and these values to create our final configuration that the application has access to. 

Open your `secrets.json` file and put your sensitive settings inside. E.g:
```json
// secrets.json

{
  "ConnectionStrings": {
    "MyConnectionString": "my-sensitive-connection-string-1234-abcd" 
  },
  // Note we didn't need the General section as this is taken from appsettings.json
}
```

## Create options classes

To access the settings during runtime we will use dependency injection of different `Options` classes, we need to make these classes.

For our examples above we will create two classes, one for each of the sections; `ConnectionStrings` and `General`:

```
Configuration/
    ConnectionStrings.cs
    General.cs
```

```csharp
// ConnectionStrings.cs

namespace MyProject.Configuration 
{
  public class ConnectionStrings 
  {
    public string RandomColour { get; set; }
    public string Shape { get; set; }
  }
}
```

```csharp
// General.cs

namespace MyProject.Configuration 
{
  public class General 
  {
    public string MyConnectionString { get; set; }
  }
}
```

## Install required packages for Dependency Injection

We need to install some NuGet packages to make sure that we will be able to inject our `Options` classes into our functions:

* Microsoft.Azure.Functions.Extensions
* Microsoft.NET.Sdk.Functions ( >= 1.0.28 )
* Microsoft.Extensions.DependencyInjections ( <= 3.x )

## Create functions setup class to configure everything

We will now create a startup class that will read values from our `appsettings.json`, our user secrets (and environment variables), merge them together into sections and then load into our preconfigured options objects. It will then finally register these options objects so our functions can request them via dependency injection. 

The class should be called `Startup.cs` and be at the root of your project. Here is an example:

```csharp
// Startup.cs

using System;
using System.IO;
using MyProject.Configuration;
using Microsoft.Azure.Functions.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

[assembly: FunctionsStartup(typeof(MyProject.Startup))]

namespace MyProject
{
    public class Startup : FunctionsStartup
    {
        private bool IsDevelopment =>
            string.Equals(Environment.GetEnvironmentVariable("AZURE_FUNCTIONS_ENVIRONMENT"), "Development", StringComparison.OrdinalIgnoreCase);

        /// <summary>
        /// Loads in settings from various sources including environment / user secrets / appsettings
        ///     and binds them to various options objects
        /// </summary>
        public override void Configure(IFunctionsHostBuilder builder)
        {
            // Bind connection strings
            builder.Services.AddOptions<ConnectionStrings>().Configure<IConfiguration>((settings, configuration) =>
            {
                configuration.GetSection(nameof(ConnectionStrings)).Bind(settings);
            });

            // Bind general settings
            builder.Services.AddOptions<General>().Configure<IConfiguration>((settings, configuration) =>
            {
                configuration.GetSection(nameof(General)).Bind(settings);
            });
        }

        /// <summary>
        /// Defines the sources in which to load application settings from so they can be used above
        /// </summary>
        public override void ConfigureAppConfiguration(IFunctionsConfigurationBuilder builder)
        {
            FunctionsHostBuilderContext context = builder.GetContext();

            builder.ConfigurationBuilder
                .AddJsonFile(Path.Combine(context.ApplicationRootPath, "appsettings.json"), optional: true);

            if (IsDevelopment)
            {
                builder.ConfigurationBuilder.AddUserSecrets<Startup>();
            } 
            else
            {
                builder.ConfigurationBuilder.AddEnvironmentVariables();
            }
        }
    }
}

```

## Accessing the settings during runtime

You now need to request the options objects via dependency injection in your function. Here is an example:

```csharp
// MyFunction.cs

...

private readonly IOptions<ConnectionStrings> _connectionStrings;
private readonly IOptions<General> _settings;

public MyFunction(IOptions<ConnectionStrings> connectionStrings, IOptions<General> settings)
{
    _connectionStrings = connectionStrings;
    _settings = settings;
}

...

// Access a setting
var shape = _settings.Value.Shape
```

## Testing the application

If you now run the functions project locally you should find values stored in either user secrets or directly in `appsettings.json` are accessible at runtime! 🎉🎉

# Deployment

User secrets are not supported when deployed, hence you need to move your sensitive values to environment variables when deployed. The values hardcoded into `appsettings.json` will still be read. You may have noticed this in `Startup.cs` that we merge from different locations depending on the environment. 

There is just once small factor you need to be made aware of. You have have to flatten your JSON when naming your environment variables. For example the following setting in our user secrets:
```json
{
  "ConnectionStrings": {
    "MyConnectionString": "my-sensitive-connection-string-1234-abcd" 
  }
}
```
becomes the following environment variable (using double underscore`__` to indicate nesting):
```
ConnectionStrings__MyConnectionString = "my-sensitive-connection-string-1234-abcd"
```

# Documentation

When a new member of your team comes to work on your project they will not have their user secrets setup correctly so their project will not run as expected. 

We suggest adding a section to your README that indicates what the layout of their `secrets.json` should be, and where they could get the value from. 

```json
// Please setup your secrets.json as follows, 
// the connection string can be found in the company password vault. 

{
  "ConnectionStrings": {
    "MyConnectionString": "<IN_PASSWORD_VAULT>" 
  }
}
```

# References

* https://docs.microsoft.com/en-us/azure/azure-functions/functions-dotnet-dependency-injection
* https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-6.0&tabs=windows
