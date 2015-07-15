### Controller Factory

#### IControllerFactory

``` csharp
using System.Web.Routing;
using System.Web.SessionState;

namespace System.Web.Mvc
{
	public interface IControllerFactory
    {
        IController CreateController(RequestContext requestContext, 
                                     string controllerName);
                                     
		SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext, 
                                                          string controllerName);
		
        void ReleaseController(IController controller);
	}
}
```

#### Instantiating Controller Classes

``` csharp
return controllerType == null 
	   ? null 
	   : (IController)DependencyResolver.Current.GetService(controllerType);
```

#### Registering a Custom Controller Factory

``` csharp
public class MvcApplication : HttpApplication
{
	protected void Application_Start() 
    {
		// ...other stuff
        
		ControllerBuilder
            .Current
            .SetControllerFactory(new CustomControllerFactory());
	}
}
```
