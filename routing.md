
===========================================================
UrlRoutingModule : IHttpModule
    PostResolveRequestCache
===========================================================

public virtual void PostResolveRequestCache(HttpContextBase context)
{
    // get route handler
    RouteData routeData = this.RouteCollection.GetRouteData(context);
    IRouteHandler routeHandler = routeData.RouteHandler;
    
    // create request context
    RequestContext requestContext = new RequestContext(context, routeData);
    context.Request.RequestContext = requestContext;
    
    // get http handler
    IHttpHandler httpHandler = routeHandler.GetHttpHandler(requestContext);
    context.RemapHandler(httpHandler);
}

    public interface IRouteHandler
    {
        IHttpHandler GetHttpHandler(RequestContext requestContext);
    }

    public interface IHttpHandler
    {
        bool IsReusable { get; }
        void ProcessRequest(HttpContext context);
    }

===========================================================
RouteCollection : Collection<RouteBase>
    GetRouteData
===========================================================

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

==========================================================
Route : RouteBase
    GetRouteData
==========================================================

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

================================================================================================================
================================================================================================================
===========================================  handler file  =====================================================
================================================================================================================
================================================================================================================

================================================
RouteCollectionExtensions
    MapRoute
================================================

public static Route MapRoute(this RouteCollection routes, string name, string url, object defaults, object constraints, string[] namespaces)
{
    Route route = new Route(url, (IRouteHandler) new MvcRouteHandler())
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

===========================================================
MvcRouteHandler : IRouteHandler
    GetHttpHandler
===========================================================

protected virtual IHttpHandler GetHttpHandler(RequestContext requestContext)
{
    requestContext.HttpContext.SetSessionStateBehavior(GetSessionStateBehavior(requestContext));
    return (IHttpHandler) new MvcHandler(requestContext);
}

============================================================
MvcHandler : IHttpHandler
    ProcessRequest
============================================================

protected virtual void ProcessRequest(HttpContext httpContext)
{
    HttpContextBase httpContextBase = new HttpContextWrapper(httpContext);
    
    IController controller;
    IControllerFactory factory;
    
    // If request validation has already been enabled, make it lazy.
    // This allows attributes like [HttpPost] (which looks at Request.Form)
    // to work correctly without triggering full validation.
    // Tolerate null HttpContext for testing.
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
        throw new InvalidOperationException("...");
    }

    try
    {
        controller.Execute(RequestContext);
    }
    finally
    {
        factory.ReleaseController(controller);
    }
}

internal ControllerBuilder ControllerBuilder
{
    get { return _controllerBuilder ?? (_controllerBuilder = ControllerBuilder.Current); }
    set { _controllerBuilder = value; }
}

========================================================
ControllerBuilder
    GetControllerFactory
========================================================
