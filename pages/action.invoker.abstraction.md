### Action Invoker

#### Abstractions

``` csharp
namespace System.Web.Mvc
{
    public interface IActionInvoker
    {
        bool InvokeAction(ControllerContext controllerContext, 
                          string actionName);
    }
}

namespace System.Web.Mvc.Async
{
    public interface IAsyncActionInvoker : IActionInvoker
    {
        IAsyncResult BeginInvokeAction(ControllerContext controllerContext, 
                                       string actionName, 
                                       AsyncCallback callback, 
                                       object state);

        bool EndInvokeAction(IAsyncResult asyncResult);
    }
}
```

### Implementing Custom Action Invoker

```csharp
public class CustomActionInvoker : IActionInvoker
{
    public bool InvokeAction(ControllerContext controllerContext,
                             string actionName)
    {
        if (actionName == "Index") 
        {
            var message = "This is output from the Index action";
            
            controllerContext.HttpContext.Response.Write(message);
            
            return true;
        }
        
        return false;
    }
}
```

### Use Custom Action Invoker

``` csharp
public class ActionInvokerController : Controller 
{
    public ActionInvokerController() 
    {
        this.ActionInvoker = new CustomActionInvoker();
    }
    
    // ...other stuff
}
```