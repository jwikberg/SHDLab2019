# SHD Azure Lab

## Prerequisites
1. [An Azure account/subscription](https://azure.microsoft.com/en-us/free). We will only use the "Always free" parts of Azure in this Lab, so don't worry if you've already used up the $200 credit for non-free services.
2. [An Azure DevOps account](https://azure.microsoft.com/en-us/services/devops/)
3. [Visual Studio Code](https://code.visualstudio.com/)
4. [.Net Core 2.2 SDK](https://dotnet.microsoft.com/download)

### VS Code Extensions
1. [C#](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)
2. [NuGet Package Manager](https://marketplace.visualstudio.com/items?itemName=jmrog.vscode-nuget-package-manager)
3. [Azure Resource Manager Tools](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools)

## 1. Azure DevOps project

Create a new private project in DevOps with a Git repository.

## 2. Generate .Net Core projects

First create the following folder structure locally:

SHDLab  
-- WebApi  
-- Tests  
-- ARM  

Use the .Net Core CLI to create a new WebApi project by navigating into the WebApi folder in your terminal/command prompt and run `dotnet new webapi` to generate the project.

Then navigate into the Tests folder and run `dotnet new xunit` to generate a project for our tests that will use xUnit.

Initilize a git repo with `git init` in the SHDLab folder then add [this .gitignore to that folder as well](https://raw.githubusercontent.com/jwikberg/SHDLab2019/master/.gitignore).  
Commit all changes.

Go to `Repos` in the DevOps project you created in step 1 and follow the directions for adding the remote and push your changes.

## 3. Build pipeline

Build pipelines in Azure DevOps can be created through the classic GUI or with yaml definitions, we will be using the latter.

Go to `Pipelines > Builds` in DevOps and create a new pipeline for your repo.

Our build pipeline is very basic and only includes steps for restoring packages, building & publishing an artifact that we later can deploy to an AppService in Azure:

```yaml
resources:
- repo: self

pool:
  name: Hosted Ubuntu 1604

steps:
- task: DotNetCoreCLI@2
  displayName: Restore
  inputs:
    command: restore
    projects: '**/*.csproj'

- task: DotNetCoreCLI@2
  displayName: Build
  inputs:
    projects: '**/*.csproj'
    arguments: '--configuration Release'

- task: DotNetCoreCLI@2
  displayName: Test
  inputs:
    command: test
    projects: '**/*[Tt]ests/*.csproj'
    arguments: '--configuration Release'

- task: DotNetCoreCLI@2
  displayName: Publish
  inputs:
    command: publish
    publishWebProjects: false
    projects: 'WebApi/*.csproj'
    arguments: '--configuration Release --output $(build.artifactstagingdirectory)'
    zipAfterPublish: true

- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact'
  inputs:
    PathtoPublish: '$(build.artifactstagingdirectory)'
  condition: succeededOrFailed()
```

## 4. Deployment using Azure Resource Manager

### 4.1 Resource Group

Go to portal.azure.com and create a new Resource group, use SHDLab as name and select West Europe as region.

### 4.2 Service Connection

To allow DevOps to deploy to and create resources in Azure, we need to create a Azure Resource Manager(ARM) service connection.

Go to `Project settings > Pipelines > Service connections` in DevOps and create a new ARM service connection.

Use DevOps-SHDLab as name for the connection and select your resource group (you'll be prompted to authenticate yourself first), then precc OK.

### 4.3 ARM Template

Our Azure resources will be created by using an ARM template.

Go to the ARM folder we created earlier, then create a file called `azuredeploy.json` with the following content:

```json
{
    "resources": [
        {
            "apiVersion": "2016-03-01",
            "name": "[parameters('name')]",
            "type": "Microsoft.Web/sites",
            "properties": {
                "name": "[parameters('name')]",
                "siteConfig": {
                    "appSettings": []
                },
                "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('hostingPlanName'))]"
            },
            "location": "[resourceGroup().location]",
            "dependsOn": [
                "[concat('Microsoft.Web/serverfarms/', parameters('hostingPlanName'))]"
            ]
        },
        {
            "apiVersion": "2016-09-01",
            "name": "[parameters('hostingPlanName')]",
            "type": "Microsoft.Web/serverfarms",
            "location": "[resourceGroup().location]",
            "properties": {
                "name": "[parameters('hostingPlanName')]",
                "workerSizeId": "0",
                "numberOfWorkers": "1"
            },
            "sku": {
                "Tier": "[parameters('sku')]",
                "Name": "[parameters('skuCode')]"
            }
        }
    ],
    "parameters": {
        "name": {
            "type": "string"
        },
        "hostingPlanName": {
            "type": "string"
        },
        "sku": {
            "type": "string"
        },
        "skuCode": {
            "type": "string"
        }
    },
    "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0"
}
```

Create another file called `azuredeploy.parameters.json` with the following content and change `YOUR_INITIALS_HERE` (this parameter will be the name of your AppService, which needs to be unique), e.g. `JW`:

```json
{
    "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "value": "SHDLab-YOUR_INITIALS_HERE"
        },
        "hostingPlanName": {
            "value": "SHDLabPlan"
        },
        "sku": {
            "value": "Free"
        },
        "skuCode": {
            "value": "F1"
        }
    }
}
```

Add the following step to the bottom of your `azure-pipelines.yml` file to publish the ARM template as an artifact that can be used in the release pipeline:

```yaml
- task: PublishBuildArtifacts@1
  displayName: 'Publish Artifact: ARM'
  inputs:
    PathtoPublish: ARM
    ArtifactName: ARM
```

Commit the new files and changes.

### 4.4 Release pipeline

Go to `Pipelines > Releases` in DevOps and create a new release pipeline. Use the `Azure App Service deployment` template.

Click on `Add an artifact` and select your build pipleine.

Click the lightning bolt icon and enable continuous deployment.

Go to the `Tasks` tab and select DevOps-SHDLab as your subscription and manually enter SHDLab-YOUR_INITIALS_HERE as the name of your App Service.

Click the + to add a new task and pick `Azure Resource Group Deployment`. Drag the new task to the top of the list.  
Pick DevOps-SHDLab as your subscription, SHDLab as your resource group and West Europe as location.  
Then select the `azuredeploy.json` file as template by first clicking the dots next to the field, then also select `azuredeploy.parameters.json` as template parameters.

Click on the `Deploy Azure App Service` step and select WebApi.zip from our linked artificats as `Package or folder`.

Save your pipeline.

## 5. Swagger documentation

By adding Swagger documentation generation for our WebApi we get an UI for testing all endpoints.

First, open `WebApi.csproj` and replace the contents with the following to add the required package and enable xml documentation generation:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netcoreapp2.2</TargetFramework>
    <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>
  </PropertyGroup>
    <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Debug|AnyCPU'">
    <DocumentationFile>bin\Debug\netcoreapp2.2\WebApi.xml</DocumentationFile>
    <PlatformTarget>AnyCPU</PlatformTarget>
  </PropertyGroup>
  <PropertyGroup Condition="'$(Configuration)|$(Platform)'=='Release|AnyCPU'">
    <DocumentationFile>bin\Release\netcoreapp2.2\WebApi.xml</DocumentationFile>
    <PlatformTarget>AnyCPU</PlatformTarget>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.App"/>
    <PackageReference Include="Microsoft.AspNetCore.Razor.Design" Version="2.2.0" PrivateAssets="All"/>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="4.0.1"/>
  </ItemGroup>
</Project>
```
Open `Startup.cs` and replace the contents with:

```cs
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.HttpsPolicy;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using Swashbuckle.AspNetCore.Swagger;

namespace WebApi
{
    public class Startup
    {
        public Startup(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public IConfiguration Configuration { get; }

        // This method gets called by the runtime. Use this method to add services to the container.
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);

            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new Info { Title = "SHD Lab", Version = "v1" });

                var filePath = Path.Combine(System.AppContext.BaseDirectory, "WebApi.xml");
                c.IncludeXmlComments(filePath);
            });
        }

        // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
        public void Configure(IApplicationBuilder app, IHostingEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }
            else
            {
                // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
                app.UseHsts();
            }

            app.UseSwagger();

            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "SHD Lab V1");
            });

            app.UseHttpsRedirection();
            app.UseMvc();
        }
    }
}
```

Commit the changes and wait for the build and release to complete, then you can head to [https://SHDLab-YOUR_INITIALS_HERE.azurewebsites.net/swagger](https://SHDLab-YOUR_INITIALS_HERE.azurewebsites.net/swagger) to see what we've accomplished so far.
