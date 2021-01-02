---
layout: post
title: Consume SharePoint Online List from Azure Functions v2.0
---

First of all, I am not a SharePoint expert, if there is any incorrect information, please let me know.

* Consume SharePoint Online List from Azure Functions v2.0
* [Consume SharePoint Online Term Store from Azure Functions v2.0](/2019/12/13/az-func-spo-termstore)
* [Webhook SharePoint Online List with Azure Functions v2.0](/2019/12/13/az-func-spo-webhook)

I faced 2 challenges when I tried to consume SPO list restful API.

1. Which packages shall I use?

    I googled a few blogs, most of them were written before 2019, which means they were using Azure Functions v1.0 (.NET Framework instead of .NET Core), so that they can add "Microsoft.SharePointOnline.CSOM" nuget package without any problems.


    However, when I added that package into my project, I received this incompatible warning:
    <img src='{{ "/public/assets/img/spo_nuget_package.png" | relative_url }}' alt="CSOM Nuget Package" />    

    Someone mentioned in Stack Overflow that <strong>TTCUE.NetCore.SharepointOnline.CSOM.16.1.8029.1200</strong> package works well on .NET Core application. It is a wrapper of Microsoft dlls that copying from Microsoft.SharePointOnline.CSOM package\lib\netcore45, I grabbed the latest dlls from Microsoft.SharePointOnline.CSOM version 16.1.19515.12000 to my lib folder and added references from there. Then the warning disappeared.

1. How to authenticate?

    Most of the related blogs were written based on the fact that Azure AD admin can create an AAD app, grant permissions to the whole instance of SPO and supply the secret. However, in our scenario, I was only allowed to get a service account (username / password) and limited to a particular collection.

    The basic authentication of SharePoint is much uglier than what I thought, it's cookie based not token. I have to call <em>GetAuthenticationCookie</em> method to generate cookie from Microsoft.SharePoint.Client.dll (the one in net45 folder - not compatible with core) as Microsoft.SharePoint.Client.Portal.dll hasn't implemented such function yet. I have to de-compile the  Microsoft.SharePoint.Client.dll and copy them to my own classes.

    ```csharp
    // Startup.cs - manage HttpClient by Polly
    var authBuilder = new SharePointAuthenticationBuilder(config);
    builder.Services.AddHttpClient("SharePoint").ConfigureHttpMessageHandlerBuilder((b) =>
            authBuilder.BuildHttpMessageHandler(b)
        )
        .SetHandlerLifetime(TimeSpan.FromMinutes(5))
        .AddPolicyHandler(GetRetryPolicy());
    ```

    ```csharp
    // SharePointAuthenticationBuilder.cs - set cookies to HttpClient
    public void BuildHttpMessageHandler(HttpMessageHandlerBuilder builder)
    {
        Uri uri = new Uri(BaseUrl);
        var credentials = new SharePointOnlineCredentials(Username, Password);
        var handler = new HttpClientHandler()
        {
            Credentials = credentials
        };
        handler.CookieContainer.SetCookies(uri, GetAuthenticationCookie(uri, true, false));

        builder.PrimaryHandler = handler;
        builder.Build();
    }
    ```

These are the necessary dlls:
* Microsoft.SharePoint.Client.Runtime.Portable
* Microsoft.SharePoint.Client.Runtime.Windows

<p class="message">
    NOTE: SharePoint is heavily relying on Windows, and so to its dlls. Thus the Azure Functions can ONLY be deployed to <strong>Windows </strong>and <strong>Microsoft.SharePoint.Client.Runtime.Windows.dll</strong> (from net45 folder) must be referenced in Azure Functions project.
</p>

Code at Github: [https://github.com/jay-coder/AZFuncSPO](https://github.com/jay-coder/AZFuncSPO)
