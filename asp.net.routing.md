### Asp.Net Routing System

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

#### UrlRoutingModule (IHttpModule)

``` csharp
namespace System.Web.Routing
{
    public class UrlRoutingModule : IHttpModule
    {
        // ... other stuff
        
        public RouteCollection RouteCollection { get; set; } = RouteTable.Routes;
        
        /*
         * Very much simplified!
         * Just the main flow extracted.
         */
        
        public virtual void PostResolveRequestCache(HttpContextBase context)
        {
            // get the route handler
            RouteData routeData = RouteCollection.GetRouteData(context);
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

#### MSDN: Key Classes for Asp.Net Routing

Source: [__MSDN | ASP.NET Routing__](https://msdn.microsoft.com/en-us/library/cc668201.aspx)

__Class__ | __Description__
----- | -----------
[__Route__](https://msdn.microsoft.com/en-us/library/system.web.routing.route.aspx) | Represents a route in a Web Forms or MVC application
[__RouteBase__](https://msdn.microsoft.com/en-us/library/system.web.routing.routebase.aspx) | Serves as the base class for all classes that represent an ASP.NET route
[__RouteTable__](https://msdn.microsoft.com/en-us/library/system.web.routing.routetable.aspx) | Stores the routes for an application
[__RouteCollection__](https://msdn.microsoft.com/en-us/library/system.web.routing.routecollection.aspx) | Provides methods that enable you to manage a collection of routes
[__RouteCollectionExtensions__](https://msdn.microsoft.com/en-us/library/system.web.mvc.routecollectionextensions.aspx) | Provides additional methods that enable you to manage a collection of routes in MVC applications
[__RouteData__](https://msdn.microsoft.com/en-us/library/system.web.routing.routedata.aspx) | Contains the values for a requested route
[__RequestContext__](https://msdn.microsoft.com/en-us/library/system.web.routing.requestcontext.aspx) | Contains information about the HTTP request that corresponds to a route
[__StopRoutingHandler__](https://msdn.microsoft.com/en-us/library/system.web.routing.stoproutinghandler.aspx) | Provides a way to specify that ASP.NET routing should not handle requests for a URL pattern
[__RouteValueDictionary__](https://msdn.microsoft.com/en-us/library/system.web.routing.routevaluedictionary.aspx) | Provides a way to store route Constraints, Defaults, and DataTokens objects
[__VirtualPathData__](https://msdn.microsoft.com/en-us/library/system.web.routing.virtualpathdata.aspx) | Provides a way to generate URLs from route information

#### MSDN: How URLs Are Matched to Routes

[__MSDN | ASP.NET Routing__](https://msdn.microsoft.com/en-us/library/cc668201.aspx): When routing handles URL requests, it tries to match the URL of the request to a route. Matching a URL request to a route depends on all the following conditions:

* The __route patterns__ that you have defined or the default route patterns, if any, that are included in your project type.
* The __order__ in which you added them to the Routes collection.
* Any __default values__ that you have provided for a route.
* Any __constraints__ that you have provided for a route.
* Whether you have defined routing to handle requests that match a physical file.

To avoid having the wrong handler handle a request, you must consider all these conditions when you define routes.
* The order in which __Route__ objects appear in the __RouteCollection__ is significant.
* Route matching is tried from the first route to the last route in the collection.
* When a match occurs, no more routes are evaluated.

In general, __add routes to the Routes property in order from the most specific route definitions to least specific ones__.

#### RouteCollection Class

Source: [__MSDN | RouteCollection Class__](https://msdn.microsoft.com/en-us/library/system.web.routing.routecollection.aspx)

The __RouteCollection__ class provides methods that enable you to manage a collection of objects that derive from the __RouteBase__ class.

__Getting the route:__
* Typically, you will use the static __Routes__ property of the __RouteTable__ class to retrieve a __RouteCollection__ object.
* The __Routes__ property stores all the routes for an ASP.NET application.
* ASP.NET routing iterates through the routes in the __Routes__ property to find the route that matches a URL.

__Getting the URL:__
* To construct a URL, you call the __.GetVirtualPath()__ method and pass in a collection of values. 
* The __.GetVirtualPath()__ method finds the first route with parameters that match the values that you passed in, and returns a __VirtualPathData__ object that contains information about the matching route. 
* You retrieve the URL through the __VirtualPath__ property of the __VirtualPathData__ object.

__Route names:__
* You can add a route either with a name or without a name.
* Including a name enables you to distinguish between similar routes when URLs are constructed. 
* If you do not specify a name, ASP.NET routing uses the first matching route in the collection to construct a URL.
* When you add an unnamed route to the __RouteCollection__ object, you cannot add a route that already is in the collection.
* When you add a named route, you cannot use a name that already identifies a route in the collection.

__Accessing the collection:__
* You use the __.GetReadLock()__ method and the __.GetWriteLock()__ method to make sure that you interact with the collection without conflicts from other processes. 

``` csharp
namespace System.Web.Routing
{
    public class RouteCollection : Collection<RouteBase>
    {
        // ... omitted stuff
        
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
    }
}
```

#### Route Class

[__MSDN | Route Class__](https://msdn.microsoft.com/en-us/library/system.web.routing.route.aspx): "*The __Route__ class enables you to specify how routing is processed in an ASP.NET application. You create a __Route__ object for each URL pattern that you want to map to a class that can handle requests that correspond to that pattern. You then add the route to the __RouteCollection__. When the application receives a request, ASP.NET routing iterates through the routes in the __RouteCollection__ to find the first route that matches the pattern of the URL.*"

Members:
* __Constraints__ - Gets or sets a dictionary of expressions that specify valid values for a URL parameter
* __DataTokens__ - Gets or sets custom values that are passed to the route handler, but which are not used to determine whether the route matches a URL pattern
* __Defaults__ - Gets or sets the values to use if the URL does not contain all the parameters
* __RouteExistingFiles__ - Gets or sets a value that indicates whether ASP.NET routing should handle URLs that match an existing file. (Inherited from RouteBase.)
* __RouteHandler__ - Gets or sets the object that processes requests for the route
* __Url__ - Gets or sets the URL pattern for the route
* __.GetRouteData()__ - Returns information about the requested route (overrides __RouteBase.GetRouteData()__)
* __.GetVirtualPath()__ - Returns information about the URL that is associated with the route (overrides __RouteBase.GetVirtualPath()__)
* __.ProcessConstraint()__ - Determines whether a parameter value matches the constraint for that parameter

``` csharp
namespace System.Web.Routing
{
    public class Route : RouteBase
    {
        // ... omitted constructor overloads

        public Route(string url, 
                     RouteValueDictionary defaults, 
                     RouteValueDictionary constraints, 
                     RouteValueDictionary dataTokens, 
                     IRouteHandler routeHandler)
        { ... }
        
        public RouteValueDictionary Constraints { get; set; }

        public RouteValueDictionary DataTokens { get; set; }

        public RouteValueDictionary Defaults { get; set; }

        public IRouteHandler RouteHandler { get; set; }

        public string Url { get { ... } set { ... } }
        
        // ... omitted stuff

        public override RouteData GetRouteData(HttpContextBase httpContext)
        {
            RouteValueDictionary values = _parsedRoute.Match(
                httpContext.Request.AppRelativeCurrentExecutionFilePath.Substring(2) + 
                httpContext.Request.PathInfo, 
                Defaults);

            RouteData routeData = new RouteData((RouteBase) this, RouteHandler);

            if (!ProcessConstraints(httpContext, 
                                    values, 
                                    RouteDirection.IncomingRequest))
                return null;

            foreach (KeyValuePair<string, object> pair in values)
                routeData.Values.Add(pair.Key, pair.Value);

            if (DataTokens != null)
                foreach (KeyValuePair<string, object> pair in DataTokens)
                    routeData.DataTokens[pair.Key] = pair.Value;

            return routeData;
        }
    }
}
```

#### RouteData Class

[__RouteData__](https://msdn.microsoft.com/en-us/library/system.web.routing.routedata.aspx) class is the container for all the information about the route requested.

Members:

* __DataTokens__ - Gets a collection of custom values that are passed to the route handler but are not used when ASP.NET routing determines whether the route matches a request
* __Route__ - Gets or sets the object that represents a route
* __RouteHandler__ - Gets or sets the object that processes a requested route
* __Values__ - Gets a collection of URL parameter values and default values for the route
* __.GetRequiredString(valueName)__ - Retrieves the value with the specified identifier

``` csharp
namespace System.Web.Routing
{
    public class RouteData
    {
        public RouteData() { }

        public RouteData(RouteBase route, IRouteHandler routeHandler) { ... }

        public RouteValueDictionary DataTokens { get; } = new RouteValueDictionary();

        public RouteValueDictionary Values { get; } = new RouteValueDictionary();

        public RouteBase Route { get; set; }

        public IRouteHandler RouteHandler { get; set; }

        public string GetRequiredString(string valueName)
        {
            object obj;
            if (Values.TryGetValue(valueName, out obj))
            {
                string str = obj as string;
                if (!string.IsNullOrEmpty(str))
                    return str;
            }

            throw new InvalidOperationException(...valueName...);
        }
    }
}
```
