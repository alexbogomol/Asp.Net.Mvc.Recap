### Default Controller Factory

#### DefaultControllerFactory class

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        public DefaultControllerFactory();
        
        public DefaultControllerFactory(IControllerActivator controllerActivator);
        
        protected internal virtual IController GetControllerInstance(RequestContext requestContext, 
                                                                     Type controllerType);
        
        protected internal virtual Type GetControllerType(RequestContext requestContext, 
                                                          string controllerName);
        
        // IControllerFactory members
        
        public virtual IController CreateController(RequestContext requestContext, 
                                                    string controllerName);
        
        protected internal virtual SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext,
                                                                                     Type controllerType);
        
        public virtual void ReleaseController(IController controller);
    }
}
```

#### Resolving the instance of IControllerActivator

**DefaultControllerFactory** has three approaches on how to obtain its controller activator:
* ctor-injected custom implementation of `IControllerActivator`
* **(?)** implemenation, provided by external `IResolver<IControllerActivator>`
* if no alternatives available, then **DefaultControllerActivator** instance will be used (provided with **SingleServiceResolver<>**)

Once controller activator instance is resolved it is internally accessed **only** through the private property **ControllerActivator**

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        // ... other stuff
        
        private IResolver<IControllerActivator> _activatorResolver;
        private IControllerActivator _controllerActivator;
        
        internal DefaultControllerFactory(IControllerActivator controllerActivator, 
                                          IResolver<IControllerActivator> activatorResolver, 
                                          IDependencyResolver dependencyResolver)
        {
            if (controllerActivator != null)
            {
                _controllerActivator = controllerActivator;
            }
            else
            {
                _activatorResolver = activatorResolver 
                                  ?? new SingleServiceResolver<IControllerActivator>(
                                             () => null,
                                             new DefaultControllerActivator(dependencyResolver),
                                             "DefaultControllerFactory constructor");
            }
        }

        private IControllerActivator ControllerActivator
        {
            get
            {
                if (_controllerActivator != null)
                {
                    return _controllerActivator;
                }
                
                _controllerActivator = _activatorResolver.Current;
                
                return _controllerActivator;
            }
        }
    }
}
```

#### IControllerFactory.CreateController()

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        // ... other stuff
        
        public virtual IController CreateController(RequestContext requestContext, string controllerName)
        {
            // security-check 'requestContext' against the null value

            if (String.IsNullOrEmpty(controllerName) && !requestContext.RouteData.HasDirectRouteMatch())
            {
                throw new ArgumentException("Null Or Empty", "controllerName");
            }

            Type controllerType = GetControllerType(requestContext, controllerName);
            
            IController controller = GetControllerInstance(requestContext, controllerType);
            
            return controller;
        }
    }
}
```

#### Get Controller Type

The process of searching the controller type is convention-based, so the controller-candidate classes **must** satisfy the following rule-set:
* *must be* __public__
* *cannot be* __abstract__
* *cannot be* __generic__
* *must end with* **"...Controller"**
* *must implement* **IController**

Decision-making steps are the folowing:

* direct route match?
    
    **`return GetControllerTypeFromDirectRoute(routeData);`**

* found in the current route's namespace collection?
    
    **`return GetControllerTypeWithinNamespaces(routeData.Route, controllerName, namespaceHash);`**

* found in the application's default namespace collection?
    
    **`return GetControllerTypeWithinNamespaces(routeData.Route, controllerName, namespaceDefaults);`**

* if all else fails, search every namespace
    
    **`return GetControllerTypeWithinNamespaces(routeData.Route, controllerName, null /* namespaces */);`**

#### Get Controller Instance

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        // ... other stuff
        
        protected internal virtual IController GetControllerInstance(RequestContext requestContext, 
                                                                     Type controllerType)
        {
            if (controllerType == null)
            {
                throw new HttpException(404, "no controller found!");
            }
            
            if (!typeof(IController).IsAssignableFrom(controllerType))
            {
                throw new ArgumentException("not an IController", "controllerType");
            }
            
            return ControllerActivator.Create(requestContext, controllerType);
        }
    }
}
```

##### todoes & future-focus

- [ ] next: **DefaultControllerActivator**
- [ ] details: **.GetControllerTypeFromDirectRoute()**
- [ ] details: **.GetControllerTypeWithinNamespaces()**
