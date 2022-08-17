---
title: "Generate a .NET 6 Web API Typescript client from an OpenAPI / Swagger definition"
date: 2022-08-17T14:53:28+01:00
draft: false
summary: "Turn your swagger.json file into a fully configured typescript API service, that you can instantly integrate into your JavaScript application"
---

### GitHub Repository

An example of this code can be found at https://github.com/dombarter/Solar.API

# Motivation

Ever wondered if there was an easy way of turning your C# DTOs into TypeScript interfaces? Every wondered if we could automatically generate a TypeScript client for our API, rather than having to manually write out the function for _every_ endpoint? Turns out the answer is yes, here's how:

# Generate The Swagger Definition

If you've recently created a .NET Web API you'll know it automatically opens up Swagger when you run the application. However it doesn't create a `swagger.json` file that is copied to your project folder.

Let's get a `swagger.json` being generated:

## Install Swashbuckle CLI

Run the following commands in your API project:

```cmd
dotnet new tool-manifest
dotnet tool install Swashbuckle.AspNetCore.Cli
```

## Add A New Post-Build Command

Add the following task to your API `csproj` which will generate a `swagger.json` file from our API build, and then copy that file into our Vue.js directory, so we can access it using `npm` commands.

```xml
<Target Name="OpenAPI" AfterTargets="Build">
    <Exec Command="dotnet tool restore" WorkingDirectory="$(ProjectDir)" />
    <Exec Command="dotnet swagger tofile --output ../Solar.Vue/references/swagger.json $(OutputPath)$(AssemblyName).dll v1" WorkingDirectory="$(ProjectDir)" />
</Target>
```

If you now build the project, you should find a `swagger.json` file under `Solar.Vue/references/`.

# Generate The TypeScript Client

Now we have our hands on a `swagger.json` we now need to build a TypeScript client. There are a couple of steps:

## Install OpenAPI Tools

Install the following `npm` package:

```cmd
npm i @openapitools/openapi-generator-cli -D
```

OpenAPI Tools is the officially supported package for everything OpenAPI/Swagger (see https://openapi.tools/)

## Install Java

The above generator requires Java to be installed. If you're on windows you can easily install it using `chocolatey`:

```cmd
choco install oraclejdk
```

## NPM Script

And finally, we need to add a `npm` script to build the client:

```cmd
"generate-api-client": "openapi-generator-cli generate -i ./references/swagger.json -g typescript-axios -o ./src/api/"
```

- `./references/swagger.json` defines where the definition is
- `./src/api` defines where the client should be created
- `typescript-axios` is a type of client (specifically written in TypeScript and using axios for http calls)

# Testing

You will now have a fully fledged TypeScript client with DTO types. Here is an example of how you can use it!

```ts
import { AccountApi } from "@/api";

const api = new AccountApi();

const result = await api.userLoginPost({
  email: "user@email.com",
  password: "password",
});
```

# References

- https://www.npmjs.com/package/@openapitools/openapi-generator-cli
- https://khalidabuhakmeh.com/generate-aspnet-core-openapi-spec-at-build-time
- https://chrlschn.medium.com/net-6-web-apis-with-openapi-typescript-client-generation-a743e7f8e4f5
- https://stackoverflow.com/questions/33283071/swagger-webapi-create-json-on-build
