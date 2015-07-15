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
            if (requestContext == null)
            {
                throw new ArgumentNullException("requestContext");
            }
            if (requestContext.HttpContext == null)
            {
                throw new ArgumentException("...", "requestContext");
            }

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