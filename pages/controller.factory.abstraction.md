### Controller Factory

#### Abstraction (IControllerFactory)

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

#### Custom IControllerFactory Implementation

``` csharp
public class CustomControllerFactory : IControllerFactory 
{
    public IController CreateController(RequestContext requestContext,
                                        string controllerName) 
    {
        Type targetType = null;
        
        // decide controller type
        switch (controllerName)
        {
            case "Default":
                targetType = typeof(DefaultController);
                break;
            case "SomeOther":
                targetType = typeof(SomeOtherController);
                break;
            default:
                // set the fallback controller type when cannot decide
                requestContext.RouteData.Values["controller"] = "Default";
                targetType = typeof(SomeOtherController);
                break;
        }
        
        // instantiate the controller class
        return targetType == null 
               ? null
               : (IController)DependencyResolver.Current.GetService(targetType);
    }

    public SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext, 
                                                             string controllerName)
    {
        return SessionStateBehavior.Default;
    }

    public void ReleaseController(IController controller) 
    {
        IDisposable disposable = controller as IDisposable;
        
        if (disposable != null) 
        {
            disposable.Dispose();
        }
    }
}
```

#### Registering a Custom Controller Factory

``` csharp
public class MvcApplication : HttpApplication
{
	protected void Application_Start() 
    {
		// ...other stuff
        
		ControllerBuilder.Current
                         .SetControllerFactory(new CustomControllerFactory());
	}
}
```
