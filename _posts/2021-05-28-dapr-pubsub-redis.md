---
layout: post
title: 'Dapr - Pub Sub Redis'
---

## Subscribe to Redis Streams

### Self-Hosted

  1. Add app.UseCloudEvents and endpoints.MapSubscribeHandler in Configure method of Startup.cs
```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    ......
    app.UseCloudEvents();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
        endpoints.MapSubscribeHandler();
    });
}
```

  2. Create a yaml file in components folder
```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    value: ""
```

  3. Create a controller for Email
```csharp
using Dapr;
using Dapr.Client;
......
namespace HTTPSendGrid.Controllers
{
    [ApiController]
    [Route("[controller]")]
    public class EmailController : ControllerBase
    {
        ......
        
        private readonly DaprClient _daprClient;
        private readonly ILogger<EmailController> _logger;

        public EmailController(ILogger<EmailController> logger, DaprClient daprClient)
        {
            _logger = logger;
            _daprClient = daprClient;
        }

        ......

        [Topic("pubsub", "emailTopic")]
        [HttpPost("PubSubSend")]
        public Task PubSubSendEmail(EmailModel email)
        {
            _logger.LogInformation($"PubSubSend Email {email.Subject} to {email.To}");
            return SendEmail(email);
        }

        ......
    }
}
```
  NOTE: pubsubName comes from the component name of yaml file at #2

  4. Run the application
```
dapr run --app-id emailservice --components-path ./components --dapr-http-port 3500 --app-port 5001 --app-ssl -- dotnet run
```

  5. Invoke the service via curl
```bash
curl --location --request POST 'http://localhost:3500/v1.0/publish/pubsub/emailTopic' \
--header 'Content-Type: application/json' \
--data-raw '{
    "from": "no-reply@test.com",
    "to": "yourname@test.com",
    "subject": "Hello Dapr SendGrid",
    "body": "<h1>Hello from Dapr PubSub</h1>"
}'
```
NOTE: Url pattern: /v1.0/publish/[pubsubName]/[topicName]

### Kubernetes
  
  1. Install Redis in Kubernetes
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis
```
  2. Check the Redis containers and get Redis password
  Windows: 
  * Will create a file with your encoded password
``` 
kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" > encoded.b64
```
  * Will put your redis password in a text file called password.txt
``` 
certutil -decode encoded.b64 password.txt
```

  3. Create a yaml file for pubsub component in k8s folder
```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-master:6379
  - name: redisPassword
    value: "GENERATE_AT_2"
```

  4. Deploy pubsub component to Kubernetes
  5. Create a Dockerfile
```
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443
```
```
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
COPY ["HTTPSendGrid.csproj", "."]
RUN dotnet restore "./HTTPSendGrid.csproj"
COPY . .
WORKDIR "/src/."
RUN dotnet build "HTTPSendGrid.csproj" -c Release -o /app/build
```
```
FROM build AS publish
RUN dotnet publish "HTTPSendGrid.csproj" -c Release -o /app/publish
```
```
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "HTTPSendGrid.dll"]
```

  6. Docker build
```
docker build -t httpsendgrid:latest .
```

  7. Create a yaml file and deploy container to Kubernetes
```bash
kind: Deployment
apiVersion: apps/v1
metadata:
  name: sendgrid-app
  namespace: default
  labels:
    app: sendgrid-app
spec:
  ......
  template:
    metadata:
      labels:
        app: sendgrid-app
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "emailservice"
        dapr.io/app-port: "80"
        dapr.io/log-level: "debug"
......
```
  8. Invoke the Kubernetes NodePort via curl
```bash
  curl --location --request POST 'http://localhost:30040/v1.0/publish/pubsub/emailTopic' \
--header 'Content-Type: application/json' \
--data-raw '{
    "from": "no-reply@test.com",
    "to": "yourname@test.com",
    "subject": "Hello Dapr SendGrid K8S",
    "body": "<h1>Hello from Dapr PubSub</h1>"
}'
```