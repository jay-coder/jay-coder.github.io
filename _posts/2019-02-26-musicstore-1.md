---
layout: post
title: MusicStore - Part1 - Init
redirect_from:
  - /musicstore-1/
---

Let's start building a platform where musicians can sell their albums online.

Code has been uploaded to github [https://github.com/jay-coder/MusicStore](https://github.com/jay-coder/MusicStore)

What technologies are we going to use:
* WebUI -  Angular 5.2 SPA application hosted on .NET Core 2.1
* WebAPI - .NET Core 2.1 API
* IdentityServer4 - .NET Core 2.1 Web Application

Identity Server 4 is an OpenID Connect and OAuth 2.0 framework which can be used to manage tokens that being used between WebUI and WebAPI.

Here is the diagram of the solution:
<img src='{{ "/public/assets/img/musicstore_diagram.png" | relative_url }}' alt="MusicStore Solution Diagram" />

1. WebUI will ask user to login
2. After login successfully, IdentityServer will issue JWT token back to WebUI
3. WebUI will consume API resources with token attached in header
4. WebAPI will talk to IdentityServer to validate token
5. If token is valid, WebAPI will return JSON data back to WebUI

## Setup WebUI project

<img src='{{ "/public/assets/img/musicstore_webui.png" | relative_url }}' alt="MusicStore Angular App" />

In Properties > launchSettings.json, update the applicationUrl port to 30002

```json
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:30002",
      "sslPort": 0
    }
  }
```

## Setup WebAPI project

<img src='{{ "/public/assets/img/musicstore_webapi.png" | relative_url }}' alt="MusicStore API App" />

In Properties > launchSettings.json, update the applicationUrl port to 30001

```json
  "iisSettings": {
    "windowsAuthentication": false, 
    "anonymousAuthentication": true, 
    "iisExpress": {
      "applicationUrl": "http://localhost:30001",
      "sslPort": 0
    }
  },
```


## Setup IdentityServer project

Clone IdentityServer from its official github:

```shell
git clone https://github.com/IdentityServer/IdentityServer4.Quickstart.UI
```

In Properties > launchSettings.json, update the applicationUrl port to 30000

```json
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:30000",
      "sslPort": 0
    }
  },
```

## Why set port over 30000

When we deploy web applications to local Kubernetes in the future, we can only access pod by port-forward to NodePort, which only allows port 30000 to 32767. To make local dev URL consistent with Kubernetes, I would set URL port between 30000 and 32767.

```shell
kubectl port-forward musicstore-project-webui-65c8bcb489-lhjcr 30002:80
```

## Run these projects

1. WebUI - should see Hello, world!
    <img src='{{ "/public/assets/img/musicstore_webui_run.png" | relative_url }}' alt="MusicStore Web UI" />

2. WebAPI - should see the default values API
    <img src='{{ "/public/assets/img/musicstore_webapi_run.png" | relative_url }}' alt="MusicStore Web API" />

3. IdentityServer - should see welcome page
    <img src='{{ "/public/assets/img/musicstore_is_run.png" | relative_url }}' alt="MusicStore Identity Server" />

    Click "here" link to login to IdentityServer, username and password can be:
    * bob / bob
    * alice / alice

    NOTE: I added test users instead of retrieving users from database as of now, I will add database support in [MusicStore - Part3 - EF to SQL Server](/2019/03/02/musicstore-3/).

    ```csharp
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddMvc();
        services.AddIdentityServer()
            .AddDeveloperSigningCredential()
            .AddInMemoryIdentityResources(Resources.GetIdentityResources())
            .AddInMemoryApiResources(Resources.GetApiResources())
            .AddInMemoryClients(Clients.Get())
            .AddTestUsers(TestUsers.Users);
    }
    ```

4. Enable Swagger to WebAPI

    Swagger is a great tool that can help developers test their Restful API quickly, for example, we can easily send Get, Post, Put, Delete request to Restful API like Postman.

      - Add Nuget Package (Swashbuckle.AspNetCore)
      - Use Swagger in Startup.cs
          ```csharp
          // Startup.cs
          public void ConfigureServices(IServiceCollection services)
          {
            ......
            // Add Swagger
            services.AddSwaggerGen(c =>
            {
              c.SwaggerDoc("v1", new Info { Title = "MusicStore API", Version = "v1" });
            });
            ......
          }

          public void Configure(IApplicationBuilder app, IHostingEnvironment env)
          {
            ......
            // Use Swagger
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
              c.SwaggerEndpoint("/swagger/v1/swagger.json", "MusicStore API V1");
            });
          }
          ```

      - Change launch URL to swagger

          ```json
          // launchSettings.json
          {
            "$schema": "http://json.schemastore.org/launchsettings.json",
            "iisSettings": {
              "windowsAuthentication": false, 
              "anonymousAuthentication": true, 
              "iisExpress": {
                "applicationUrl": "http://localhost:30001",
                "sslPort": 0
              }
            },
            "profiles": {
              "IIS Express": {
                "commandName": "IISExpress",
                "launchBrowser": true,
                "launchUrl": "swagger",
                "environmentVariables": {
                  "ASPNETCORE_ENVIRONMENT": "Development"
                }
              },
              "WebAPI": {
                "commandName": "Project",
                "launchBrowser": true,
                "launchUrl": "swagger",
                "applicationUrl": "http://localhost:5001",
                "environmentVariables": {
                  "ASPNETCORE_ENVIRONMENT": "Development"
                }
              }
            }
          }
          ```

    Swagger UI once configured correctly
    <img src='{{ "/public/assets/img/musicstore_swagger_ui.png" | relative_url }}' alt="MusicStore Swagger UI" />