### DefaultControllerFactory class

#### Controller classes requirements by DefaultControllerFactory

* The class must be public.
* The class must be concrete (not abstract).
* The class must not take generic parameters.
* The name of the class must end with **...Controller**.
* The class must implement the `IController` interface.

#### DefaultControllerFactory

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
