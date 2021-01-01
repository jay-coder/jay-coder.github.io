---
layout: post
title: Sitecore - fieldNameTranslator null exception
---

After finishing the secondment to innovation team, I had to re-configure my local Sitecore environment and upgrade SXA from 1.3 to 1.7 which had been done by others when I was away.

After getting the latest code from VSTS and restoring the database, I got this welcome message:
{% highlight csharp %}
Server Error in '/' Application. 

A route named 'MS_attributerouteWebApi' is already in the route collection. Route names must be unique. Parameter name: name Description: An unhandled exception occurred during the execution of the current web request. Please review the stack trace for more information about the error and where it originated in the code.

Exception Details: System.ArgumentException: A route named 'MS_attributerouteWebApi' is already in the route collection. Route names must be unique. Parameter name: name

Source Error:

An unhandled exception was generated during the execution of the current web request. Information regarding the origin and location of the exception can be identified using the exception stack trace below.

Stack Trace:

[ArgumentException: A route named 'MS_attributerouteWebApi' is already in the route collection. Route names must be unique. Parameter name: name] System.Web.Routing.RouteCollection.Add(String name, RouteBase item) +391
System.Web.Http.Routing.AttributeRoutingMapper.MapAttributeRoutes(HttpConfiguration configuration, IInlineConstraintResolver constraintResolver, IDirectRouteProvider directRouteProvider) +166
Sitecore.Services.Infrastructure.Web.Http.HttpConfigurationBuilder.MapHttpAttributeRoutes() +89 Sitecore.Services.Infrastructure.Web.Http.ServicesConfigurator.Configure(HttpConfiguration config, RouteCollection routes) +501
Sitecore.Services.Infrastructure.Sitecore.Pipelines.ServicesWebApiInitializer.Process(PipelineArgs args) +194 (Object , Object[] ) +74
Sitecore.Pipelines.CorePipeline.Run(PipelineArgs args) +469
Sitecore.Pipelines.DefaultCorePipelineManager.Run(String pipelineName, PipelineArgs args, String pipelineDomain) +22
Sitecore.Nexus.Web.HttpModule.Application_Start() +161
Sitecore.Nexus.Web.HttpModule.Init(HttpApplication app) +764
System.Web.HttpApplication.RegisterEventSubscriptionsWithIIS(IntPtr appContext, HttpContext context, MethodInfo[] handlers) +581
System.Web.HttpApplication.InitSpecial(HttpApplicationState state, MethodInfo[] handlers, IntPtr appContext, HttpContext context) +172
System.Web.HttpApplicationFactory.GetSpecialApplicationInstance(IntPtr appContext, HttpContext context) +418
System.Web.Hosting.PipelineRuntime.InitializeApplication(IntPtr appContext) +369

[HttpException (0x80004005): A route named 'MS_attributerouteWebApi' is already in the route collection. Route names must be unique. Parameter name: name]
System.Web.HttpRuntime.FirstRequestInit(HttpContext context) +534
System.Web.HttpRuntime.EnsureFirstRequestInit(HttpContext context) +111 System.Web.HttpRuntime.ProcessRequestNotificationPrivate(IIS7WorkerRequest wr, HttpContext context) +718
{% endhighlight %}

Luckily, I got WinMerge. After comparing the working version, I realized someone accidentally created a <strong>global.asax</strong> and overwrote the default one.

Then I managed to login by deleting that asax file. However, when I clicked on tree nodes in Content Editor, I got exception again:

{% highlight csharp %}
Server Error in '/' Application.

Value cannot be null.
Parameter name: fieldNameTranslator
Description: An unhandled exception occurred during the execution of the current web request. Please review the stack trace for more information about the error and where it originated in the code. 

Exception Details: System.ArgumentNullException: Value cannot be null.
Parameter name: fieldNameTranslator

Source Error: 

An unhandled exception was generated during the execution of the current web request. Information regarding the origin and location of the exception can be identified using the exception stack trace below.

Stack Trace: 
[ArgumentNullException: Value cannot be null.
Parameter name: fieldNameTranslator]
   Sitecore.ContentSearch.Linq.Solr.SolrIndexParameters..ctor(IIndexValueFormatter valueFormatter, IFieldQueryTranslatorMap`1 fieldQueryTranslators, FieldNameTranslator fieldNameTranslator, IExecutionContext[] executionContexts, IFieldMapReaders fieldMap, Boolean convertQueryDatesToUtc) +328
   Sitecore.ContentSearch.SolrProvider.LinqToSolrIndex`1..ctor(SolrSearchContext context, IExecutionContext[] executionContexts) +188
   Sitecore.ContentSearch.SolrProvider.SolrSearchContext.GetQueryable(IExecutionContext[] executionContexts) +268
   Sitecore.Social.Search.SearchProvider.SearchItems(Expression`1 whereExpression, Func`2 selector) +164
   Sitecore.Social.Search.SearchProvider.GetMessagesByContainer(String container) +492
   Sitecore.Social.MessageBusinessManager.SearchMessagesByContainer(String container) +203
   Sitecore.Social.MessageBusinessManager.GetMessagesCount(String container) +16
   Sitecore.Social.Client.MessagePosting.Commands.SocialCenter.RunGetHeader(CommandContext context, String header) +342
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.FillParamsFromCommand(CommandContext commandContext, RibbonCommandParams ribbonCommandParams) +98
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.GetCommandParameters(Item controlItem, CommandContext commandContext) +79
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderLargeButton(HtmlTextWriter output, Item button, CommandContext commandContext) +78
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderButton(HtmlTextWriter output, Item button, CommandContext commandContext) +440
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderChunk(HtmlTextWriter output, Item chunk, CommandContext commandContext) +342
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderChunk(HtmlTextWriter output, Item chunk, CommandContext commandContext, Boolean isContextual, String id) +244
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderChunk(HtmlTextWriter output, Item chunk, CommandContext commandContext, Boolean isContextual) +161
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderChunks(HtmlTextWriter output, Item strip, CommandContext commandContext, Boolean isContextual) +445
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderStrips(HtmlTextWriter output, Item ribbon, Boolean isContextual, ListString visibleStripList) +1613
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.RenderStrips(HtmlTextWriter output, Item defaultRibbon, Item contextualRibbon, ListString visibleStripList) +162
   Sitecore.Web.UI.WebControls.Ribbons.Ribbon.Render(HtmlTextWriter output) +747
   System.Web.UI.Control.RenderControlInternal(HtmlTextWriter writer, ControlAdapter adapter) +79
   Sitecore.Web.HtmlUtil.RenderControl(Control ctl) +80
   Sitecore.Shell.Applications.ContentManager.ContentEditorForm.UpdateRibbon(Item folder, Boolean isCurrentItemChanged, Boolean showEditor) +502
   Sitecore.Shell.Applications.ContentManager.ContentEditorForm.Update() +582
   Sitecore.Shell.Applications.ContentManager.ContentEditorForm.OnPreRendered(EventArgs e) +205

[TargetInvocationException: Exception has been thrown by the target of an invocation.]
   System.RuntimeMethodHandle.InvokeMethod(Object target, Object[] arguments, Signature sig, Boolean constructor) +0
   System.Reflection.RuntimeMethodInfo.UnsafeInvokeInternal(Object obj, Object[] parameters, Object[] arguments) +128
   System.Reflection.RuntimeMethodInfo.Invoke(Object obj, BindingFlags invokeAttr, Binder binder, Object[] parameters, CultureInfo culture) +142
   Sitecore.Reflection.ReflectionUtil.InvokeMethod(MethodInfo method, Object[] parameters, Object obj) +89
   Sitecore.Shell.Applications.ContentManager.ContentEditorPage.OnPreRender(EventArgs e) +143
   System.Web.UI.Control.PreRenderRecursiveInternal() +162
   System.Web.UI.Page.ProcessRequestMain(Boolean includeStagesBeforeAsyncPoint, Boolean includeStagesAfterAsyncPoint) +6875
{% endhighlight %}

And some warnings from crawling log:
{% highlight csharp %}
18828 22:59:24 WARN  Failed to initialize 'sitecore_marketing_asset_index_master' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_marketing_asset_index_web' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_marketingdefinitions_master' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_marketingdefinitions_web' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_testing_index' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_suggested_test_index' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_fxm_master_index' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_fxm_web_index' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'sitecore_list_index' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'social_messages_master' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
18828 22:59:24 WARN  Failed to initialize 'social_messages_web' index. Registering the index for re-initialization once connection to SOLR becomes available ...
18828 22:59:24 WARN  DONE
{% endhighlight %}

It's because I didn't create those indexes yet:
* sitecore_marketing_asset_index_master
* sitecore_marketing_asset_index_web
* ......

Finally, Sitecore started working after adding those indexes in Solr configuration, hooray!