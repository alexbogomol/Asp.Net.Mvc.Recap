### Controller

#### Abstractions

``` csharp
using System.Web.Routing;

namespace System.Web.Mvc
{
    public interface IController
    {
        void Execute(RequestContext requestContext);
    }
}

namespace System.Web.Mvc.Async
{
    public interface IAsyncController : IController
    {
        IAsyncResult BeginExecute(RequestContext requestContext, 
                                  AsyncCallback callback, 
                                  object state);
        
        void EndExecute(IAsyncResult asyncResult);
    }
}
```

#### Custom IController Implementation

This is how we can implement our own `IController`. This implementation substitutes almost all the framework functionality, because it short-cuts the concept of actions|results|filters and process of action-decision. So we can use our own implementations for super-specific cotrollers. The standard approach is to derive the existing abstract **Controller**.

``` csharp
public class BasicController : IController 
{
    public void Execute(RequestContext requestContext)
    {
        string controller = (string)requestContext.RouteData.Values["controller"];
        string action     = (string)requestContext.RouteData.Values["action"];

        if (action == "redirect") 
        {
            var url = "some/other/location";
            requestContext.HttpContext.Response.Redirect(url);
            return;
        } 
        
        var message = string.Format("Controller: {0}, Action: {1}", controller, action);
        requestContext.HttpContext.Response.Write(message);
    }
}
```

#### ControllerBase class

``` csharp
namespace System.Web.Mvc
{
    public abstract class ControllerBase : IController
    {
        void IController.Execute(RequestContext requestContext)
        {
            Execute(requestContext);
        }

        protected virtual void Execute(RequestContext requestContext)
        {
            // security-check 'requestContext' against the null value
            // security-check 'requestContext.HttpContext' against the null value

            VerifyExecuteCalledOnce();
            Initialize(requestContext);

            using (ScopeStorage.CreateTransientScope())
            {
                ExecuteCore();
            }
        }

        protected abstract void ExecuteCore();

        protected virtual void Initialize(RequestContext requestContext)
        {
            ControllerContext = new ControllerContext(requestContext, this);
        }
        
        public ControllerContext ControllerContext { get; set; }

        public TempDataDictionary TempData { get { ... } set { ... } }

        public bool ValidateRequest { get { ... } set { ... } }

        public IValueProvider ValueProvider { get { ... } set { ... } }

        public dynamic ViewBag { get { ... } }

        public ViewDataDictionary ViewData { get { ... } set { ... } }
    }
}
```

#### Controller

``` csharp
namespace System.Web.Mvc
{
    public abstract class Controller : ControllerBase, IAsyncController, ...
    {
        // ... other stuff
        
        protected override void ExecuteCore()
        {
            PossiblyLoadTempData();
            
            try
            {
                string actionName = GetActionName(RouteData);
                
                if (!ActionInvoker.InvokeAction(ControllerContext, actionName))
                {
                    HandleUnknownAction(actionName);
                }
            }
            finally
            {
                PossiblySaveTempData();
            }
        }
        
        // IAsyncController implementation
        
        IAsyncResult IAsyncController.BeginExecute(RequestContext requestContext, 
                                                   AsyncCallback callback, 
                                                   object state)
        {
            return BeginExecute(requestContext, callback, state);
        }

        void IAsyncController.EndExecute(IAsyncResult asyncResult)
        {
            EndExecute(asyncResult);
        }
    }
}
```
