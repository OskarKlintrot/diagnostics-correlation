# Microsoft.Diagnostics.Correlation
The Correlation library allows web applications to correlate events through several tires communicating over HTTP(s). It allows to:
>- extract correlation-ids from incoming request
>- inject correlation-ids to outgoing request
>- keep correlation context though request lifecycle
### Correlation Identifiers
Library provides functionality to works with three identifiers
>- **correlation id** common identifier of the whole operation (or flow) potentially involving multiple HTTP calls. By default is extracted from ```x-ms-request-root-id``` header and generated if not found. 
>- **request id**: identifier of this particular request. By default is extracted from ```x-ms-request-id```, ASP.NET ```TraceIdentifier``` is used if not set.
>- **child request id** identifies outgoing request id, passed to downstream service in  ```x-ms-request-id``` and becomes request-id there. Generated by the library.

> **Notes:**
Use ```CorrelationHeaderInfo``` to change header names
## Nuget Packages
>-  **Microsoft.Diagnostics.Correlation.AspNetCore**
Supported on NET 4.X. and .NET Core. This is standalone package which should be used for ASP.NET Core apps.
>- **Microsoft.Diagnostics.Correlation.AspNet**
Supported on	.NET 4.X. Provides ActionFilters for MVC and WebApi on top of Correlation package (below).
>-  **Microsoft.Diagnostics.Correlation**
Supported on	.NET 4.X
This package alone should be used for non ASP.NET apps or ASP.NET apps hosted with OWIN self-host.

> **Notes:**
>- You may use **Microsoft.Diagnostics.Mvc** and/or **Microsoft.Diagnostics.WebApi** packages instead of **Microsoft.Diagnostics.Correlation.AspNet**
>- **Microsoft.Diagnostics.Instrumentation** inplements profiler instrumentation. Currently does not ship as nuget package.

## Incoming request parsing
### ASP.NET Apps
#### OWIN Middleware
>- ```Microsoft.Diagnostics.Correlation.Common.Owin.CorrelationContextTracingMiddleware``` Middleware extracts correlation-id (string) and request-id (string) from HTTP request headers and stores them in ```CorrelationContext```
>- ```Microsoft.Diagnostics.Correlation.Common.Owin.ContextTracingMiddleware<TContext>``` Generic middleware allowing to specify ```IContextFactory``` to generate any context from the request headers.
> **Note:**
In OWIN self-hosted applications, middleware provides all the necessary capabilities to parse incoming request and store it's context.
It may be substituted with MVC or WebApi ActionFilter provided by the suite, however it would mean that other Owin middleware(s) used by application, e.g. for logging, will not be able to use advantages of the correlation context. 
We strongly recommend to have ```CorrelationContextTracingMiddleware``` or ```ContextTracingMiddleware``` the first middleware invoked.

#### ASP.NET apps hosted in IIS
Due to ASP.NET thread agility model, context filled by the OWIN middleware will not be retained, so the suit provides ```ActionFilter``` for WebApi and MVC to set the context or restore the lost one.
So, application can still use middleware to parse request and log it, however ActionFilter should also be configured to use correlation context in application.

>- ```Microsoft.Diagnostics.Correlation.WebApi.CorrelationTracingFilter``` 
ActionFilter for WebApi controllers
>- ```Microsoft.Diagnostics.Correlation.Mvc.CorrelationTracingFilter``` 
ActionFilter for MVC controllers

We recommend to add above filters (as appropriate) to the global filters collection rather than adding attribute to every controller.
> **Note:**
Please note, filters work only with ```CorrelationContext```.

### ASP.NET Core apps
Correlation listens to diagnostics events from ASP.NET Core. Use `app.UseCorrelationInstrumentation` method to enable it.

## Injecting outgoing requests
When one service calls another service in the multi-tier application, the suite allows to inject header to outgoing request.
The suite provides several approaches for it:
### ```DelegatingHandler``` in HttpClient pipeline 
Library provides:
```Microsoft.Diagnostics.Correlation.Common.Http.CorrelationContextRequestHandler``` ```DelegatingHandler``` which could be integrated in existing pipeline. Developers also could use builders provided by the library to construct ```HttpClient``` with ```CorrelationContextRequestHandler```.

```HttpClientBuilder``` and ```CorrelationHttpClientBuilder``` allow to provide inner handler along with other delegating handlers to build custom pipeline, however ```ContextRequestHandler<TContext>``` will be called first.

>- ```Microsoft.Diagnostics.Correlation.Http.CorrelationHttpClientBuilder```  provides builder for ```CorrelationContext```
>- ```Microsoft.Diagnostics.Correlation.Http.HttpClientBuilder``` provides generic way to specify how context should be injected in the request 

> **Note:**
>- ```CorrelationContextRequestHandler``` only works with ```CorrelationContext```. Use ```ContextRequestHandler<TContext>``` for other context types.
>- We recommend creating ```HttpClient``` for every dependency you have and use dependency injection framework to make ```HttpClient``` instances available to controllers and retain them though the application lifecycle

#### Logging outgoing requests
It's important to log every outgoing request. Usually, for  ```HttpClient``` it's done with  ```DelegatingHandler```, however library does not provide such handler and expect users to add it to the pipeline.
Outgoing request includes ```child request id``` passed as header for the downstream service, so when the request is logged, child request id should also be logged as a part of the context. 
However, ```CorrelationContext``` is tight to ```incoming request``` rather than to outgoing request and there could be multiple outgoing requests having different ```child request ids```.
To get the child context, use one of 2 APIs:
>-  ```CorrelationContext.GetChildRequestContext```. Please note, it will clone 'parent' context
>- Log 'parent' context and ```child request id``` separately with extension method ```HttpRequestMessage.GetChildRequestId()```

>**Notes:**
> Child request id is stored in the request, so as long as you have reference to ```HttpRequestMessage```, you may properly log it. 

### HttpClient instrumentation in .NET Core
```HttpClient``` is instrumented  in .NET Core with [DiagnosticSource](https://docs.microsoft.com/en-us/dotnet/core/api/system.diagnostics.diagnosticsource). It allows to intercept HttpClient calls and instrument outgoing requests. You may enable it with: 
`app.UseCorrelationInstrumentation(Configuration.GetSection("Correlation"))`, making sure `InstrumentOutgoingRequests` is set to `false`
**Note:**
You may pass your own `IContextFactory` and list of `IContextInjector` by using low level `Microsoft.Diagnostics.Correlation.AspNetCore.ContextTracingInstrumentation.Enable<TContext>` method

> **Note:**
> Make sure your implementations of ```IContextFactory``` and ```IContextInjector``` do not have complex logic, or do any time-consuming operations. They will be called for any outgoing and incoming request and therefore may have heavy performance impact.

#### Logging outgoing requests
Unlike ```HttpClient``` handler pipeline, ```DiagnosticListener``` instrumentation does not allow to hook into it, so the library notifies about outgoing request events with ```IOutgoingRequestNotifier``` 
Context received in ```IOutgoingRequestNotifier```  already contains ```child request id``` 
You may implement ```IOutgoingRequestNotifier``` and register it in application services.

### Profiler (does not have nuget package)
Library provides capability to instrument ```WebRequest.BeginGetResponse``` (which would be called for ```HttpClient``` and ```WebRequest``` methods). You may enable it with
>- ```Microsoft.Diagnostics.Correlation.Instrumentation.ContextTracingInstrumentation.Enable```
>- ```Microsoft.Diagnostics.Correlation.Instrumentation.ContextTracingInstrumentation.Enable<TContext>``` - allows to specify ```IContextFactory``` to generate any context from the request and collection of ```IContextInjector``` to inject correlation context to  outgoing request.
Profiler requires some configuration to be done with application: agent should be installed on the host and app should run with some environment variables.
1. For Azure Web Applications, just install ApplicationInsights Extension on the portal
2. For CloudServices, please follow AI help: https://azure.microsoft.com/en-us/documentation/articles/app-insights-cloudservices/

> **Note:**
To use profiler together with ApplicationInsights, please diable runtime instrumentation in the applicationInsights.config:
 ```<Add Type="Microsoft.ApplicationInsights.DependencyCollector.DependencyTrackingTelemetryModule, Microsoft.AI.DependencyCollector">
      <DisableRuntimeInstrumentation>true</DisableRuntimeInstrumentation>
</Add>```

#### Logging outgoing requests
Unlike ```HttpClient``` handler pipeline, Profiler instrumentation does not allow to hook into it, so the library notifies about outgoing request events with ```IOutgoingRequestNotifier``` 
Context received in ```IOutgoingRequestNotifier```  already contains ```child request id``` 
You may configure ```IOutgoingRequestNotifier``` in ```Configuration``` class and pass to ```Enable``` methods.

## Accessing context
```CorrelationContext``` or other custom context implemented by developer may be accessed while request is still being processed with ```ContextResolver``` class
```ContextResolver``` is based on [```AsyncLocal```](https://msdn.microsoft.com/en-us/library/dn906268%28v=vs.110%29.aspx) or [```CallContext```](https://msdn.microsoft.com/en-us/library/system.runtime.remoting.messaging.callcontext%28v=vs.110%29.aspx) depending on the target framework.
> **Note:**
Context is stored as ```object```

Correlation library provides implementation of ```CorrelationContext``` and several ```IContextFactory```  and ```IContextInjector``` which work with it.
```CorrelationContext``` is dictionary, containing correlation-id and request-id and any other custom field which could be added or removed.


## Samples
### ASP.NET Core
You can define correlation configuration in the settings file or construct `AspNetCoreCorrelationConfiguration`.
Sample configuration:
```
"Correlation" : {
    "InstrumentOutgoingRequests" : true,
    "Headers" : {
      "CorrelationIdHeaderName" : "x-custom-correlation-id",
      "RequestIdHeaderName" : "x-custom-request-id",
    },
    "EndpointFilter" : {
        "Allow" : false,
        "Endpoints" : ["core\.windows\.net", "dc\.services\.visualstudio\.com"]
    }
}
```
All parameters are optional.
- `InstrumentOutgoingRequests` - enables/disables outgoing requests instrumentation, true by default,
- `Headers` - allows to set correlaiton header names. Default values are `x-ms-request-root-id` (correlation-id) and `x-ms-request-id` (request id)
- `EndpointFilter` - filter for outgoing request `Uri`, allows to control which doutgoing requests are instrumented. By default rejects instrumentation for Azure Storage and ApplicationInsights endpoints,


#### HttpClient handler
In ```Startup.cs```

>- Create necessary ```HttpClient``` instances and register them:
```
    services.AddSingleton(CorrelationHttpClientBuilder.CreateClient());
```
>- Enable correlation support:
```
   app.UseCorrelationInstrumentation(Configuration.GetSection("Correlation"));
```
#### DiagnosticSource Instrumentation
>- Register `IOutgoingRequestNotifier` to receive outgoing requests notifications:
```
    services.AddRequestNotifier(new OutgoingRequestNotifier());
```
>- Enable correlation support:
```
   app.UseCorrelationInstrumentation(Configuration.GetSection("Correlation"));
```

### ASP.NET
You may use OWIN correlation middleware or HttpModule implementation to handle incoming requests,
however MVC or WebAPI filter should be configured in ```Global.asax```
##### **ActionFilters**
>- **MVC**:
```
   GlobalFilters.Filters.Add(new Microsoft.Diagnostics.Correlation.Mvc.CorrelationTracingFilter());
   FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
```
>- **WebAPI**
```
    GlobalConfiguration.Configuration.Filters.Add(new Microsoft.Diagnostics.Correlation.WebApi.CorrelationTracingFilter());
```
##### **HttpModule**
Add ```CorrelationTracingHttpModule``` 
```
<add name="CorrelationTracingHttpModule" type="Microsoft.Diagnostics.Correlation.Http.CorrelationTracingHttpModule, Microsoft.Diagnostics.Correlation" preCondition="managedHandler" />
```
> **Note**:
HttpModule should be added under 
>- ```<system.web>/<httpModules>``` for IIS 6.0 
>- ```<system.webServer>/<modules>``` for IIS 7.0
>
### OWIN Self-Hosted apps
```
app.Use<CorrelationContextTracingMiddleware>();
```
### ApplicationInsights integration
If you use ApplicationInsights to collect telemetry data, you need to configure ```TelemetryInitializer``` to map ```CorrelationContext``` fields to AppInsights properties:
```
    public class CorrelationTelemetryInitializer : ITelemetryInitializer
    {
        public void Initialize(ITelemetry telemetry)
        {
            //add request id to every event
            var ctx = ContextResolver.GetRequestContext<CorrelationContext>();
            if (ctx != null)
            {
                telemetry.Context.Operation.Id = ctx.CorrelationId;
                telemetry.Context.Operation.ParentId = ctx.RequestId;
            }
        }
    }
```
>- **Note**
```child-request-id``` currently cannot be added to dependency telemetry