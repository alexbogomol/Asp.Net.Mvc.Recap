### Routing

#### Asp.Net Abstractions

``` csharp
public interface IRouteHandler
{
    IHttpHandler GetHttpHandler(RequestContext requestContext);
}

public interface IHttpHandler
{
    bool IsReusable { get; }
    
    void ProcessRequest(HttpContext context);
}
```

#### UrlRoutingModule : IHttpModule

``` csharp
namespace System.Web.Routing
{
    public class UrlRoutingModule : IHttpModule
    {
        // ... other stuff
        
        public virtual void PostResolveRequestCache(HttpContextBase context)
        {
            // get the route handler
            RouteData routeData = this.RouteCollection.GetRouteData(context);
            IRouteHandler routeHandler = routeData.RouteHandler;
            
            // create request context
            RequestContext requestContext = new RequestContext(context, routeData);
            context.Request.RequestContext = requestContext;
            
            // get the http handler
            IHttpHandler httpHandler = routeHandler.GetHttpHandler(requestContext);
            context.RemapHandler(httpHandler);
        }
    }
}
```

#### RouteCollection : Collection<RouteBase>

``` csharp
public RouteData GetRouteData(HttpContextBase httpContext)
{
    foreach (RouteBase routeBase in (Collection<RouteBase>) this)
    {
        RouteData routeData = routeBase.GetRouteData(httpContext);
        if (routeData != null)
        {
            return routeData;
        }
    }
    return null;
}
```

#### Route (RouteBase)

``` csharp
public override RouteData GetRouteData(HttpContextBase httpContext)
{
    RouteValueDictionary values = _parsedRoute.Match(
        httpContext.Request.AppRelativeCurrentExecutionFilePath.Substring(2) + httpContext.Request.PathInfo, 
        Defaults);

    RouteData routeData = new RouteData((RouteBase) this, this.RouteHandler);

    if (!ProcessConstraints(httpContext, values, RouteDirection.IncomingRequest))
        return null;

    foreach (KeyValuePair<string, object> pair in values)
        routeData.Values.Add(pair.Key, pair.Value);

    if (DataTokens != null)
        foreach (KeyValuePair<string, object> pair in this.DataTokens)
            routeData.DataTokens[pair.Key] = pair.Value;

    return routeData;
}
```

#### MapRoute to RouteCollection

``` csharp
namespace System.Web.Mvc
{
    public static class RouteCollectionExtensions
    {
        // ... other staff
        
        public static Route MapRoute(this RouteCollection routes, 
                                     string name, 
                                     string url, 
                                     object defaults, 
                                     object constraints, 
                                     string[] namespaces)
        {
            // ... check 'route' and 'url' against null values
            
            Route route = new Route(url, new MvcRouteHandler())
            {
                Defaults = CreateRouteValueDictionaryUncached(defaults),
                Constraints = CreateRouteValueDictionaryUncached(constraints),
                DataTokens = new RouteValueDictionary()
            };

            ConstraintValidation.Validate(route);

            if ((namespaces != null) && (namespaces.Length > 0))
            {
                route.DataTokens[RouteDataTokenKeys.Namespaces] = namespaces;
            }

            routes.Add(name, route);

            return route;
        }
    }
}
```

#### MvcRouteHandler (IRouteHandler)

**MvcRouteHandler** issues a new instance of **MvcHandler** after setting the appropriate **SessionStateBehavior**. **ControllerFactory** is used to provide the **SessionStateBehavior**, and can be provided with custom constructor, otherwise the current from **ControllerBuilder** will be taken.

``` csharp
namespace System.Web.Mvc
{
    public class MvcRouteHandler : IRouteHandler
    {
        IHttpHandler IRouteHandler.GetHttpHandler(RequestContext requestContext)
        {
            return GetHttpHandler(requestContext);
        }
        
        private IControllerFactory _controllerFactory;

        public MvcRouteHandler() { }

        public MvcRouteHandler(IControllerFactory controllerFactory)
        {
            _controllerFactory = controllerFactory;
        }

        protected virtual IHttpHandler GetHttpHandler(RequestContext requestContext)
        {
            requestContext.HttpContext.SetSessionStateBehavior(GetSessionStateBehavior(requestContext));
            
            return new MvcHandler(requestContext);
        }

        protected virtual SessionStateBehavior GetSessionStateBehavior(RequestContext requestContext)
        {
            string controllerName = (string)requestContext.RouteData.Values["controller"];
            
            if (String.IsNullOrWhiteSpace(controllerName))
            {
                throw new InvalidOperationException("Route Values Has No Controller");
            }

            IControllerFactory controllerFactory = 
                              _controllerFactory ?? ControllerBuilder.Current.GetControllerFactory();
            
            return controllerFactory.GetControllerSessionBehavior(requestContext, controllerName);
        }
    }
}
```

#### MvcHandler (IHttpHandler)

``` csharp
namespace System.Web.Mvc
{
    public class MvcHandler : IHttpHandler, ...
    {
        bool IHttpHandler.IsReusable
        {
            get { return IsReusable; }
        }

        void IHttpHandler.ProcessRequest(HttpContext httpContext)
        {
            ProcessRequest(httpContext);
        }
        
        public MvcHandler(RequestContext requestContext)
        {
            if (requestContext == null)
            {
                throw new ArgumentNullException("requestContext");
            }

            RequestContext = requestContext;
        }

        internal ControllerBuilder ControllerBuilder { get { ... } set { ... } }

        protected virtual bool IsReusable
        {
            get { return false; }
        }

        public RequestContext RequestContext { get; private set; }
        
        protected virtual void ProcessRequest(HttpContext httpContext)
        {
            HttpContextBase httpContextBase = new HttpContextWrapper(httpContext);
            
            ProcessRequest(httpContextBase);
        }

        protected internal virtual void ProcessRequest(HttpContextBase httpContext)
        {
            IController controller;
            IControllerFactory factory;
            ProcessRequestInit(httpContext, out controller, out factory);

            try
            {
                controller.Execute(RequestContext);
            }
            finally
            {
                factory.ReleaseController(controller);
            }
        }

        private void ProcessRequestInit(HttpContextBase httpContext, 
                                        out IController controller, 
                                        out IControllerFactory factory)
        {
            /*
             * If request validation has already been enabled, make it lazy. 
             * This allows attributes like [HttpPost] (which looks at Request.Form) 
             * to work correctly without triggering full validation.
             * Tolerate null HttpContext for testing.
             */
             
            HttpContext currentContext = HttpContext.Current;
            if (currentContext != null)
            {
                bool? isRequestValidationEnabled = ValidationUtility.IsValidationEnabled(currentContext);
                if (isRequestValidationEnabled == true)
                {
                    ValidationUtility.EnableDynamicValidation(currentContext);
                }
            }

            AddVersionHeader(httpContext);
            RemoveOptionalRoutingParameters();

            // Get the controller type
            string controllerName = RequestContext.RouteData.GetRequiredString("controller");

            // Instantiate the controller and call Execute
            factory = ControllerBuilder.GetControllerFactory();
            controller = factory.CreateController(RequestContext, controllerName);
            if (controller == null)
            {
                throw new InvalidOperationException("Factory Returned Null");
            }
        }
    }
}
```