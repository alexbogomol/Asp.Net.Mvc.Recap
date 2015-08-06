### Mvc Routing

#### MapRoute to RouteCollection

__RouteCollectionExtensions__ provides a set extention-method for convenience purposes to interact with the __RouteCollection__.

The __.MapRoute()__ methods help us to register a new route in __RouteCollection__:
* creates a new __MvcRouteHandler__ (__`IRouteHandler`__)
* creates a new __Route__, mapping the url-pattern provided to the handler just created
* supplies the created route with parameters: defaults, constraints, datatokens
* adds the route to the __RouteCollection__
* issues the route in return

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
            // ... omitted stuff
            
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
