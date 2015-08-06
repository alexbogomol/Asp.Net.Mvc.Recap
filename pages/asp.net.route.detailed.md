### Route source (decompiled)

``` csharp
namespace System.Web.Routing
{
    public class Route : RouteBase
    {
        private ParsedRoute _parsedRoute;

        public RouteValueDictionary Constraints { get; set; }

        public RouteValueDictionary DataTokens { get; set; }

        public RouteValueDictionary Defaults { get; set; }

        public IRouteHandler RouteHandler { get; set; }

        public string Url
        {
            get { return _url ?? string.Empty; }
            set
            {
                _parsedRoute = RouteParser.Parse(value);
                _url = value;
            }
        }

        public Route(string url, 
                     RouteValueDictionary defaults, 
                     RouteValueDictionary constraints, 
                     RouteValueDictionary dataTokens, 
                     IRouteHandler routeHandler)
        { .. }

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

        public override VirtualPathData GetVirtualPath(RequestContext requestContext, 
                                                       RouteValueDictionary values)
        {
            BoundUrl boundUrl = _parsedRoute.Bind(requestContext.RouteData.Values, 
                                                  values, 
                                                  Defaults, 
                                                  Constraints);

            if (!ProcessConstraints(requestContext.HttpContext, 
                                    boundUrl.Values, 
                                    RouteDirection.UrlGeneration))
                return null;

            VirtualPathData virtualPathData = new VirtualPathData((RouteBase) this, boundUrl.Url);

            if (DataTokens != null)
                foreach (KeyValuePair<string, object> keyValuePair in DataTokens)
                    virtualPathData.DataTokens[keyValuePair.Key] = keyValuePair.Value;

            return virtualPathData;
        }

        protected virtual bool ProcessConstraint(HttpContextBase httpContext, 
                                                 object constraint, 
                                                 string parameterName, 
                                                 RouteValueDictionary values, 
                                                 RouteDirection routeDirection)
        {
            IRouteConstraint routeConstraint = constraint as IRouteConstraint;

            if (routeConstraint != null)
                return routeConstraint.Match(httpContext, 
                                             this, 
                                             parameterName, 
                                             values, 
                                             routeDirection);

            string str = constraint as string;

            if (str == null)
                throw new InvalidOperationException("Must Be String Or Custom Constraint");
            
            object obj;
            
            values.TryGetValue(parameterName, out obj);
            
            return Regex.IsMatch(Convert.ToString(obj, CultureInfo.InvariantCulture), 
                                 "^(" + str + ")$", 
                                 RegexOptions.IgnoreCase | RegexOptions.CultureInvariant);
        }

        private bool ProcessConstraints(HttpContextBase httpContext, 
                                        RouteValueDictionary values, 
                                        RouteDirection routeDirection)
        {
            if (Constraints == null) return true;

            foreach (KeyValuePair<string, object> pair in Constraints)
            {
                if (!ProcessConstraint(httpContext, 
                                       pair.Value, 
                                       pair.Key, 
                                       values, 
                                       routeDirection))
                    return false;
            }
        }
    }
}
```
