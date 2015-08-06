### Asp.Net Lifecycle

This is a brief listing of call-stack steps of Asp.Net application from the viewpoint of Mvc Framework

```
HTTP Request  => | HttpApplication.Application_Start
                 | HttpApplication.Init
                 | IHttpModule.Init (for each registered module)
                 | HttpApplication.PostResolveRequestCache
                 | UrlRoutingModule.PostResolveRequestCache
                 | HttpApplication.BeginProcessRequest
                 |      MvcHandler.ProcessRequest
Http Response <= | HttpApplication.EndProcessRequest
```
