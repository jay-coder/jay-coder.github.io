---
layout: post
title: Webhook SharePoint Online List with Azure Functions v2.0
redirect_from:
  - /az-func-spo-webhook/
---

* [Consume SharePoint Online List from Azure Functions v2.0](/2019/12/13/az-func-spo-list)
* [Consume SharePoint Online Term Store from Azure Functions v2.0](/2019/12/13/az-func-spo-termstore)
* Webhook SharePoint Online List with Azure Functions v2.0

In order to register a webhook on SPO list, I need to subscribe from that list.  For example: 

```csharp
POST /_api/web/lists('5C77031A-9621-4DFC-BB5D-57803A94E91D')/subscriptions
Accept: application/json
Content-Type: application/json

{
  "resource": "https://contoso.sharepoint.com/_api/web/lists('5C77031A-9621-4DFC-BB5D-57803A94E91D')",
  "notificationUrl": "https://91e383a5.ngrok.io/api/webhook/handlerequest",
  "expirationDateTime": "2016-04-27T16:17:57+00:00"
}
```

How can I trigger this post without having the access token? I don't have permission to get the details of AAD,  so it's impossible to acquire access token and attach to the http post request using Postman.

Then I found a useful Chrome extension called "[SP Editor](https://chrome.google.com/webstore/detail/sp-editor/ecblfcmjnbbgaojblcpmjoamegpbodhd?hl=en)" which can list, create or delete webhooks by sharing my SharePoint cookies in browser. It's much easier to manage webhooks this way.

I got anther obstacle when I tried to add one simple hook. It gave me this error "Failed to validate the notification URL". In my mind, as long as my function returned 200, the subscription should be able to register. However, it turned out that SharePoint will check if the destination Url can return the same token before creating subscription.

I just updated my code with one more "if" and return the validation token at the beginning of function.

```csharp
[FunctionName("SharePointWebhook")]
public async Task<IActionResult> SharePointWebhook(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = null)] HttpRequest req,
    ILogger log)
{
    // Validation from Webhook of SharePoint
    string validationToken = req.Query["validationtoken"];
    if (!string.IsNullOrEmpty(validationToken))
    {
        return new OkObjectResult(validationToken);
    }
    ......            
}
```

Code at Github: [https://github.com/jay-coder/AZFuncSPO](https://github.com/jay-coder/AZFuncSPO)