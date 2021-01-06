---
layout: post
title: Consume SharePoint Online Term Store from Azure Functions v2.0
redirect_from:
  - /az-func-spo-termstore/
---

* [Consume SharePoint Online List from Azure Functions v2.0](/2019/12/13/az-func-spo-list)
* Consume SharePoint Online Term Store from Azure Functions v2.0
* [Webhook SharePoint Online List with Azure Functions v2.0](/2019/12/13/az-func-spo-webhook)

I had been shocked by SharePoint that not all the components have restful API yet, Term Store is one of them. There is a dodgy way to consume Term Store via list called <strong>TaxonomyHiddenList</strong>, however, the JSON returned by API only contains one field "Path" that can be utilized to generate tree. Moreover, Path is combined by title (not ID), like "Parent Title:Title:ChildTitle". Because title is not a unique key, it can cause issue to build the wrong hierarchy if titles are the same.

After discussing with our SharePoint expert, he could build a web part in SharePoint to convert the taxonomy to list, but there is no way to trigger the web part automatically in the latest version of SPO, the platform has been upgraded to use React JS rather than the classic web control now. So, there is no better way than calling the "beautiful" CSOM dll, and it's the only way I can retrieve Term's parent to build the hierarchy.

Fortunately, Microsoft supplied the portal dll for taxonomy in netcore45, I just need to include those 2 dlls: Microsoft.SharePoint.Client.Taxonomy.Portable.dll Microsoft.SharePoint.Client.Portable.dll

```csharp
// SharePointTermStore.cs - get Terms by ClientContext
using (var ctx = new ClientContext(collectionUrl))
{
    ctx.Credentials = new SharePointOnlineCredentials(username, password);
    var taxonomySession = TaxonomySession.GetTaxonomySession(ctx);
    var termStore = taxonomySession.GetDefaultSiteCollectionTermStore();
    ctx.Load(termStore,
        s => s.Groups.Include(
            g => g.TermSets.Include(
                ts => ts.Name
            )
        )
    );
    await ctx.ExecuteQueryAsync();
    _termStore = termStore;
}
```

At last, I managed to get the correct descendants of Term Store, even with "Custom Sort Order" enabled.

Code at Github: [https://github.com/jay-coder/AZFuncSPO](https://github.com/jay-coder/AZFuncSPO)
