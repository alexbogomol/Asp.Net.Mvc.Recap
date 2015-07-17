### Default Controller Factory

#### Controller classes requirements by DefaultControllerFactory

* The class must be public.
* The class must be concrete (not abstract).
* The class must not take generic parameters.
* The name of the class must end with **...Controller**.
* The class must implement the `IController` interface.

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
        
        // IControllerFactory
        public virtual IController CreateController(RequestContext requestContext, 
                                                    string controllerName);
        protected internal virtual SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext,
                                                                                     Type controllerType);
        public virtual void ReleaseController(IController controller);
    }
}
```

#### Resolving the instance of IControllerActivator

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        // ... other stuff
        
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