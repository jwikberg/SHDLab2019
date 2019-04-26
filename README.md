# SHD Azure Lab

## Prerequisites
1. [An Azure account/subscription](https://azure.microsoft.com/en-us/free). We will only use the "Always free" parts of Azure in this Lab, so don't worry if you've already used up the $200 credit for non-free services.
2. [An Azure DevOps account](https://azure.microsoft.com/en-us/services/devops/)
3. [Visual Studio Code](https://code.visualstudio.com/)
4. [.Net Core 2.2 SDK](https://dotnet.microsoft.com/download)

### VS Code Extensions
1. [C#](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)
2. [Azure Resource Manager Tools](https://marketplace.visualstudio.com/items?itemName=msazurermtools.azurerm-vscode-tools)

## 1. Azure DevOps project

Create a new private project in DevOps with a Git repository.

## 2. Generate .Net Core projects

First create the following folder structure locally:

SHDLab  
-- WebApi  
-- WebApp  
-- Tests  
-- ARM  

Use the .Net Core CLI to create a new WebApi project by navigating into the WebApi folder in your terminal/command prompt and run `dotnet new webapi` to generate the project.

Navigate into the WebApp folder and run `dotnet new webapp` to generate a web app project that we'll use to 

Then navigate into the Tests folder and run `dotnet new xunit` to generate a project for our tests that will use xUnit.

Initilize a git repo with `git init` in the SHDLab folder then add [this .gitignore to that folder as well](https://raw.githubusercontent.com/jwikberg/SHDLab2019/master/.gitignore).  
Commit all changes.

Go to `Repos` in the DevOps project you created in step 1 and follow the directions for adding the remote and push your changes.

## 3. Build pipeline

Build pipelines in Azure DevOps can be created through the classic GUI or with yaml definitions, we will be using the latter.

Go to `Pipelines > Builds` in DevOps and create a new pipeline for your repo.

Our build pipeline is very basic and only includes steps for restoring packages, building, testing & publishing the artifacts that we later can deploy to Azure:

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
  displayName: Publish App
  inputs:
    command: publish
    publishWebProjects: true
    projects: 'WebApp/*.csproj'
    arguments: '--configuration Release --output $(build.artifactstagingdirectory)'
    zipAfterPublish: true

- task: DotNetCoreCLI@2
  displayName: Publish Api
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

Use DevOps-SHDLab as name for the connection and select your resource group (you'll be prompted to authenticate yourself first), then press OK.

### 4.3 ARM Template

Our Azure resources will be created by using an ARM template.

Go to the ARM folder we created earlier, then create a file called `azuredeploy.json` with the following content:

```json
{
    "resources": [
        {
            "apiVersion": "2016-03-01",
            "name": "[parameters('appName')]",
            "type": "Microsoft.Web/sites",
            "properties": {
                "name": "[parameters('appName')]",
                "siteConfig": {
                    "appSettings": [
                        {
                            "name": "ApiName",
                            "value": "[variables('apiName')]"
                        },
                        {
                            "name": "ApplicationInsights:InstrumentationKey",
                            "value": "[reference(concat('microsoft.insights/components/', parameters('appName'))).InstrumentationKey]"
                        }
                    ]
                },
                "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', parameters('appHostingPlanName'))]"
            },
            "location": "[resourceGroup().location]",
            "resources": [
                {
                    "apiVersion": "2015-08-01",
                    "name": "Microsoft.ApplicationInsights.AzureWebSites",
                    "type": "siteextensions",
                    "dependsOn": [
                        "[resourceId('Microsoft.Web/Sites', parameters('appName'))]"
                    ],
                    "properties": {
                    }
                }
            ],
            "dependsOn": [
                "[concat('Microsoft.Web/serverfarms/', parameters('appHostingPlanName'))]",
                "[resourceId('microsoft.insights/components/', parameters('appName'))]"
            ]
        },
        {
            "type": "microsoft.insights/components",
            "apiVersion": "2015-05-01",
            "name": "[parameters('appName')]",
            "location": "westeurope",
            "tags": {
                "applicationType": "web"
            },
            "kind": "web",
            "properties": {
                "Application_Type": "web",
                "Flow_Type": "Redfield",
                "Request_Source": "AppServiceEnablementCreate"
            }
        },
        {
            "apiVersion": "2016-09-01",
            "name": "[parameters('appHostingPlanName')]",
            "type": "Microsoft.Web/serverfarms",
            "location": "[resourceGroup().location]",
            "properties": {
                "name": "[parameters('appHostingPlanName')]",
                "workerSizeId": "0",
                "numberOfWorkers": "1"
            },
            "sku": {
                "Tier": "[parameters('sku')]",
                "Name": "[parameters('skuCode')]"
            }
        },
        {
            "apiVersion": "2016-03-01",
            "name": "[variables('apiName')]",
            "type": "Microsoft.Web/sites",
            "properties": {
                "name": "[variables('apiName')]",
                "siteConfig": {
                    "appSettings": [
                        {
                            "name": "ApplicationInsights:InstrumentationKey",
                            "value": "[reference(concat('microsoft.insights/components/', variables('apiName'))).InstrumentationKey]"
                        }
                    ]
                },
                "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('apiHostingPlanName'))]"
            },
            "location": "[resourceGroup().location]",
            "resources": [
                {
                    "apiVersion": "2015-08-01",
                    "name": "Microsoft.ApplicationInsights.AzureWebSites",
                    "type": "siteextensions",
                    "dependsOn": [
                        "[resourceId('Microsoft.Web/Sites', variables('apiName'))]"
                    ],
                    "properties": {
                    }
                }
            ],
            "dependsOn": [
                "[concat('Microsoft.Web/serverfarms/', variables('apiHostingPlanName'))]",
                "[resourceId('microsoft.insights/components/', variables('apiName'))]"
            ]
        },
        {
            "type": "Microsoft.Insights/components",
            "apiVersion": "2015-05-01",
            "name": "[variables('apiName')]",
            "location": "westeurope",
            "tags": {
                "applicationType": "web"
            },
            "kind": "web",
            "properties": {
                "Application_Type": "web",
                "Flow_Type": "Redfield",
                "Request_Source": "AppServiceEnablementCreate"
            }
        },
        {
            "apiVersion": "2016-09-01",
            "name": "[variables('apiHostingPlanName')]",
            "type": "Microsoft.Web/serverfarms",
            "location": "[resourceGroup().location]",
            "properties": {
                "name": "[variables('apiHostingPlanName')]",
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
        "appName": {
            "type": "string"
        },
        "appHostingPlanName": {
            "type": "string"
        },
        "sku": {
            "type": "string"
        },
        "skuCode": {
            "type": "string"
        }
    },
    "variables": {
        "apiName": "[concat(parameters('appName'), '-api')]",
        "apiHostingPlanName": "[concat(parameters('appHostingPlanName'), 'Api')]"
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
        "appName": {
            "value": "SHDLab-YOUR_INITIALS_HERE"
        },
        "appHostingPlanName": {
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

Go to `Pipelines > Releases` in DevOps and create a new release pipeline. Click `Empty job` at the top of the right blade.

Click on `Add an artifact` and select your build pipleine.

Click the lightning bolt icon and enable continuous deployment.

Go to the `Tasks` tab, click the + to add a new task and pick `Azure Resource Group Deployment`.  
Pick `DevOps-SHDLab` as your subscription, `SHDLab` as your resource group and `West Europe` as location.  
Then select the `azuredeploy.json` file as template by first clicking the dots next to the field, then also select `azuredeploy.parameters.json` as template parameters.

Add another new task and select `Azure App Service Deploy`.  
Select `DevOps-SHDLab` as your subscription and manually enter `SHDLab-YOUR_INITIALS_HERE-api` as App Service name.
Then select `WebApi.zip` from our linked artificats as Package or folder.

Right click on the app service deploy task and clone it.  
Change App Service name to `SHDLab-YOUR_INITIALS_HERE` and select `WebApp.zip` as package.

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

Commit the changes and wait for the build and release to complete, then you can head to [https://SHDLab-YOUR_INITIALS_HERE-api.azurewebsites.net/swagger](https://SHDLab-YOUR_INITIALS_HERE-api.azurewebsites.net/swagger) to see what we've accomplished so far.

## 6. Application Insights

>Azure Application Insights provides in-depth monitoring of your web application down to the code level.  
>You can easily monitor your web application for availability, performance, and usage.  
>You can also quickly identify and diagnose errors in your application without waiting for a user to report them.

To add Application Insights to our projects, you'll first need to add the following package reference to `WebApi.csproj` and `WebApp.csproj`:

```xml
<PackageReference Include="Microsoft.ApplicationInsights.AspNetCore" Version="2.6.1"/>
```

Then open `appsettings.json` in the WebApp and WebApi projects and add the following:

```json
"ApplicationInsights": {
  "InstrumentationKey": ""
}
```

Finally, open `program.cs` in both projects as well and replace the `CreateWebHostBuilder` method with the following:

```cs
public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        .UseApplicationInsights()
        .UseStartup<Startup>();
```

Commit and push your changes.

## 7. Adding a new API controller

Create a new folder called `Models` to both the `WebApi` and `WebApp` projects and create a new file/class in both of them called `TodoItem.cs` and paste this:

```cs
namespace WebApi.Models
{
    public class TodoItem
    {
        public long Id { get; set; }
        public string Name { get; set; }
        public bool IsComplete { get; set; }
    }
}
```

Replace `namespace WebApi.Models` with `namespace WebApp.Models` in the web app model.

Create a new file/class called `TodoContext.cs` in the Models folder of the `WebApi` project:

```cs
using Microsoft.EntityFrameworkCore;

namespace WebApi.Models
{
    public class TodoContext : DbContext
    {
        public TodoContext(DbContextOptions<TodoContext> options)
            : base(options)
        {
        }

        public DbSet<TodoItem> TodoItems { get; set; }
    }
}
```

Open `Startup.cs` in the `WebApi` project and add the following above `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);`:

```cs
services.AddDbContext<TodoContext>(opt =>
    opt.UseInMemoryDatabase("TodoList"));
```

Also add the following usings to that class as well:
```cs
using Microsoft.EntityFrameworkCore;
using WebApi.Models;
```

Rename `ValuesController.cs` to `TodosController.cs` and replace the contents with the following:

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using WebApi.Models;

namespace WebApi.Controllers
{
    [Produces("application/json")]
    [Route("api/[controller]")]
    [ApiController]
    public class TodosController : ControllerBase
    {
        private readonly TodoContext _context;
        
        public TodosController(TodoContext context)
        {
            _context = context;

            if (_context.TodoItems.Count() == 0)
            {
                // Create a new TodoItem if collection is empty,
                // which means you can't delete all TodoItems.
                _context.TodoItems.Add(new TodoItem { Name = "Item1" });
                _context.SaveChanges();
            }
        }
        
        // GET: api/Todo
        [HttpGet]
        public async Task<ActionResult<IEnumerable<TodoItem>>> GetTodoItems()
        {
            return await _context.TodoItems.ToListAsync();
        }

        // GET: api/Todo/5
        [HttpGet("{id}")]
        public async Task<ActionResult<TodoItem>> GetTodoItem(long id)
        {
            var todoItem = await _context.TodoItems.FindAsync(id);

            if (todoItem == null)
            {
                return NotFound();
            }

            return todoItem;
        }

        // POST: api/Todo
        [HttpPost]
        public async Task<ActionResult<TodoItem>> PostTodoItem(TodoItem item)
        {
            _context.TodoItems.Add(item);
            await _context.SaveChangesAsync();

            return CreatedAtAction(nameof(GetTodoItem), new { id = item.Id }, item);
        }

        // PUT: api/Todo/5
        [HttpPut("{id}")]
        public async Task<IActionResult> PutTodoItem(long id, TodoItem item)
        {
            if (id != item.Id)
            {
                return BadRequest();
            }

            _context.Entry(item).State = EntityState.Modified;
            await _context.SaveChangesAsync();

            return NoContent();
        }

        // DELETE: api/Todo/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> DeleteTodoItem(long id)
        {
            var todoItem = await _context.TodoItems.FindAsync(id);

            if (todoItem == null)
            {
                return NotFound();
            }

            _context.TodoItems.Remove(todoItem);
            await _context.SaveChangesAsync();

            return NoContent();
        }
    }
}
```

Open `Index.cshtml.cs` in the web app and replace the contents with:

```cs
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using System.Net.Http;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;
using Microsoft.Extensions.Configuration;
using Newtonsoft.Json;
using WebApp.Models;

namespace WebApp.Pages
{
    public class IndexModel : PageModel
    {
        private readonly IHttpClientFactory _clientFactory;
        private readonly string _apiBaseUrl;

        public IndexModel(IHttpClientFactory clientFactory, IConfiguration configuration)
        {
            _clientFactory = clientFactory;
            _apiBaseUrl = "https://" + configuration.GetValue<string>("ApiName") + ".azurewebsites.net";
        }

        public List<TodoItem> TodoItems = new List<TodoItem>();

        public async Task OnGetAsync()
        {
            var client = _clientFactory.CreateClient();
            var response = await client.GetAsync(_apiBaseUrl + "/api/Todos");
            TodoItems = await response.Content.ReadAsAsync<List<TodoItem>>();
        }

        public async Task<IActionResult> OnGetDeleteAsync(int id) 
        {
            Console.WriteLine("ID:" + id);
            var client = _clientFactory.CreateClient();
            await client.DeleteAsync(_apiBaseUrl + "/api/Todos/" + id);

            return RedirectToPage();
        }
    }
}
```

Then open `Index.cshtml` and replace the contents with:

```cs
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="text-center">
    <h1 class="display-4">Welcome</h1>
    <p>Learn about <a href="https://docs.microsoft.com/aspnet/core">building Web apps with ASP.NET Core</a>.</p>
</div>
<h1>Todo Items</h1>
<div class="col-6">
    @foreach (var todoItem in Model.TodoItems)
    {
        <ul class="list-group">
            <li class="list-group-item">
                <p>Name: @todoItem.Name</p>
                <p>Complete: @todoItem.IsComplete</p>
                <a class="btn btn-primary" role="button" asp-page-handler="Delete" asp-route-id="@todoItem.Id">Delete</a>
            </li>
        </ul>
    }
</div>
```

Finally, open `Startup.cs` in the web app and the following below `services.AddMvc().SetCompatibilityVersion(CompatibilityVersion.Version_2_2);`:

```cs
services.AddHttpClient();
```
