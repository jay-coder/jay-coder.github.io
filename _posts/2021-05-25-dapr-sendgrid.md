---
layout: post
title: 'Dapr - SendGrid'
---

## Send email via SendGrid

  1. Create a new .NET Core WebAPI project
  ```
  dotnet new webapi -n HTTPSendGrid
  ```

  2. Install Dapr nuget package
  ```
  dotnet add package Dapr.AspNetCore
  ```

  3. Append AddDapr() in ConfigureServices method of Startup.cs
  ```csharp
  public void ConfigureServices(IServiceCollection services)
  {
      services.AddControllers().AddDapr();
      services.AddSwaggerGen(c =>
      {
          c.SwaggerDoc("v1", new OpenApiInfo { Title = "HTTPSendGrid", Version = "v1" });
      });
  }
  ```

  4. Create a yaml file in components folder
  ```
  apiVersion: dapr.io/v1alpha1
  kind: Component
  metadata:
    name: sendgrid
    namespace: default
  spec:
    type: bindings.twilio.sendgrid
    version: v1
    metadata:
    - name: apiKey
      value: YOUR_SENDGRID_KEY
  ```
  <p class="message">NOTE: Dapr is initialized to <strong>%USERPROFILE%\.dapr\</strong></p>

  5. Create a controller for Email
  ```csharp
  using Dapr;
  using Dapr.Client;
  using HTTPSendGrid.Models;
  using Microsoft.AspNetCore.Mvc;
  using Microsoft.Extensions.Logging;
  using System.Collections.Generic;
  using System.Threading.Tasks;

  namespace HTTPSendGrid.Controllers
  {
      [ApiController]
      [Route("[controller]")]
      public class EmailController : ControllerBase
      {
          private const string SendMailBinding = "sendgrid"; // The same as component name in yaml file at #4
          private const string CreateBindingOperation = "create";

          
          private readonly DaprClient _daprClient;
          private readonly ILogger<EmailController> _logger;

          public EmailController(ILogger<EmailController> logger, DaprClient daprClient)
          {
              _logger = logger;
              _daprClient = daprClient;
          }

          [HttpPost("HttpSend")]
          public Task HttpSendEmail([FromBody] EmailModel email)
          {
              _logger.LogInformation($"HttpSend Email {email.Subject} to {email.To}");
              return SendEmail(email);
          }

          private Task SendEmail(EmailModel email)
          {
              var emailDic = new Dictionary<string, string>
              {
                  ["emailFrom"] = email.From,
                  ["emailTo"] = email.To,
                  ["subject"] = email.Subject
              };
              if (!string.IsNullOrEmpty(email.CC))
              {
                  emailDic["emailCc"] = email.CC;
              }
              if (!string.IsNullOrEmpty(email.BCC))
              {
                  emailDic["emailBcc"] = email.BCC;
              }
              return _daprClient.InvokeBindingAsync(
                  SendMailBinding,
                  CreateBindingOperation,
                  email.Body,
                  emailDic
              );
          }
      }
  }
  ```

  6. Run the application
  ```
    dapr run --app-id emailservice --components-path ./components --dapr-http-port 3500 --app-port 5001 --app-ssl -- dotnet run
  ```
  <img src='{{ "/public/assets/img/dapr_run_command.png" | relative_url }}' alt="Dapr Run Command" />

  7. Invoke the service via curl
  ```bash
    curl --location --request POST 'http://localhost:3500/v1.0/invoke/emailservice/method/email/httpsend' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "from": "no-reply@test.com",
        "to": "yourname@test.com",
        "subject": "Hello Dapr SendGrid",
        "body": "<h1>Hello from HTTP Request</h1>"
    }'
  ```

  <p class="message">NOTE: Url pattern: /v1.0/invoke/[--app-id]/method/[controller]/[method route]</p>