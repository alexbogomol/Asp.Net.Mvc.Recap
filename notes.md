## Notes

### Controller classes requirements by DefaultControllerFactory

* The class must be public.
* The class must be concrete (not abstract).
* The class must not take generic parameters.
* The name of the class must end with **...Controller**.
* The class must implement the `IController` interface.

### DefaultControllerFactory

``` csharp
namespace System.Web.Mvc
{
    public class DefaultControllerFactory : IControllerFactory
    {
        public DefaultControllerFactory();
        public DefaultControllerFactory(IControllerActivator controllerActivator);
        
        protected internal virtual IController GetControllerInstance(RequestContext requestContext, Type controllerType);
        protected internal virtual Type GetControllerType(RequestContext requestContext, string controllerName);
        
        // IControllerFactory
        public virtual IController CreateController(RequestContext requestContext, string controllerName);
        protected internal virtual SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext, Type controllerType);
        public virtual void ReleaseController(IController controller);
    }
}
```

### Registering a Custom Controller Factory

``` csharp
public class MvcApplication : HttpApplication {

	protected void Application_Start() {
		...
		ControllerBuilder
            .Current
            .SetControllerFactory(new CustomControllerFactory());
	}
}
```

### Instantiating Controller Classes

``` csharp
return targetType == null 
	   ? null 
	   : (IController)DependencyResolver.Current.GetService(targetType);
```

### IControllerFactory

``` csharp
using System.Web.Routing;
using System.Web.SessionState;

namespace System.Web.Mvc {
	public interface IControllerFactory {
		IController CreateController(RequestContext requestContext, string controllerName);
		SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext, string controllerName);
		void ReleaseController(IController controller);
	}
}
```