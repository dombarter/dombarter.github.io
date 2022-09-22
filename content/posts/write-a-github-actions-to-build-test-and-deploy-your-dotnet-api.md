---
title: "Write a GitHub Actions workflow to build, test and deploy your .NET 6 API to Azure App Service"
date: 2022-09-22T16:11:07+01:00
draft: false
summary: "Learn how to create a CI/CD pipeline that will check and deploy your changes everytime you push to main"
---

# Motivation

As part of my dissertation project at university I needed a simple CI/CD pipeline to continually test and deploy my API to Azure.

# Setup

Go to your project and create a new folder called `.github`, with another directory within called `workflows`. Inside this folder you want to create a `yml` file. I decided to name mine after my Azure resource:

```cmd
ase-vertigo-prod-uksouth.yml
```

Now you need to paste in this basic workflow:

```yml
# Action name
name: ase-vertigo-prod-uksouth

# Branch triggers
on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  ci:
    runs-on: windows-latest

    steps:
      # Checkout git repository
      - uses: actions/checkout@v2
```

- The `name` will show up in `GitHub` to show you which action you are running
- The `on` section defines triggers for the workflow: on push to `main` and a `manual` trigger
- And finally we will run on a `Windows` machine, and checkout the repository

# Restore NuGet packages

Add the following step to restore all NuGet packages in the solution:

```yml
# Restore NuGet packages
- name: Restore
  run: dotnet restore
```

# Build and publish the API project

Add the following step to build the API project and publish it to a set directory:

```yml
# Build and publish the API project
- name: Publish
  run: |
    cd ${{ env.API_PROJECT }}
    dotnet build --configuration Release
    dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/myapp
```

# Test the project

Add the following step to run a `xUnit` test project (in our case, some API integration tests):

```yml
# Test the API project
- name: Test
  run: |
    cd ${{ env.API_TEST_PROJECT }}
    dotnet test
```

# Deploy to Azure App Service

Add the following code to take the published bundle and push it to Azure App Service:

```yml
# Deploy to Azure
- uses: azure/webapps-deploy@v2
  name: Deploy
  with:
    app-name: ${{ env.AZURE_WEBAPP_NAME }}
    publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
    package: ${{env.DOTNET_ROOT}}/myapp
```

# Environment variables

We need to add some environment variables at the top so our workflow knows which projects to build and what the Azure app service is called:

```yml
# Environment variables
env:
  API_PROJECT: Vertigo.API
  API_TEST_PROJECT: Vertigo.API.Test
  AZURE_WEBAPP_NAME: ase-vertigo-prod-uksouth
```

You can also add any extra environment variables if they are needed by the test project for example.

# Configure Azure publish profile

Finally we need to get a private setting called an `AZURE_PUBLISH_PROFILE` from our app service to create an authenticated deployment.

Follow the steps here: https://learn.microsoft.com/en-us/dotnet/devops/dotnet-publish-github-action#add-publish-profile

# Final steps

The workflow should then appear automatically in GitHub, and run on one of the triggers:

![GitHub Actions](/images/github-actions.png)

# Complete Workflow

Here is the complete workflow with all the changes:

```yml
# Action name
name: ase-vertigo-prod-uksouth

# Branch triggers
on:
  push:
    branches:
      - main
  workflow_dispatch:

# Environment variables
env:
  API_PROJECT: Vertigo.API
  API_TEST_PROJECT: Vertigo.API.Test
  AZURE_WEBAPP_NAME: ase-vertigo-prod-uksouth

jobs:
  ci:
    runs-on: windows-latest

    steps:
      # Checkout git repository
      - uses: actions/checkout@v2

      # Restore NuGet packages
      - name: Restore
        run: dotnet restore

      # Build and publish the API project
      - name: Publish
        run: |
          cd ${{ env.API_PROJECT }}
          dotnet build --configuration Release
          dotnet publish -c Release -o ${{env.DOTNET_ROOT}}/myapp

      # Test the API project
      - name: Test
        run: |
          cd ${{ env.API_TEST_PROJECT }}
          dotnet test

      # Deploy to Azure
      - uses: azure/webapps-deploy@v2
        name: Deploy
        with:
          app-name: ${{ env.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
          package: ${{env.DOTNET_ROOT}}/myapp
```

# References

https://learn.microsoft.com/en-us/dotnet/devops/dotnet-publish-github-action
