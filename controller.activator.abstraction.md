### Controller Activator

#### Abstraction (IControllerActivator)

``` csharp
using System.Web.Routing;

namespace System.Web.Mvc
{
    public interface IControllerActivator
    {
        IController Create(RequestContext requestContext, Type controllerType);
    }
}
```

#### Implementing Custom Controller Activator

Here we implement custom `IControllerActivator` which intercepts all calls to **ProductController** and issues an instance of **CustomerController** in return.

``` csharp
public class CustomControllerActivator : IControllerActivator
{
    public IController Create(RequestContext requestContext, Type controllerType)
    {
        if (controllerType == typeof(ProductController))
        {
            controllerType = typeof(CustomerController);
        }
        
        return (IController)DependencyResolver.Current.GetService(controllerType);
    }
}
```

#### Registering Custom Controller Activator

`IControllerActivator` is an infrastructure element of **DefaultControllerFactory**, hence we instruct **ControllerBuilder** to use **DefaultControllerFactory** with ctor-injected instance of **CustomControllerActivator**.

``` csharp
public class MvcApplication : HttpApplication
{
    protected void Application_Start()
    {
        // ... other stuff
        
        ControllerBuilder.Current
                         .SetControllerFactory(
                             new DefaultControllerFactory(
                                 new CustomControllerActivator()));
    }
}
```
