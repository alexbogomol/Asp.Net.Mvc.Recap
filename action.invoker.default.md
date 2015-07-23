### Built-In (default) Action Invoker

#### Built-In ControllerActionInvoker

__ControllerActionInvoker__ is the basic (legacy) implementation for `IActionInvoker` in Asp.Net.Mvc framework.

``` csharp
namespace System.Web.Mvc
{
    public class ControllerActionInvoker : IActionInvoker
    {
        // ... other stuff
        
        public virtual bool InvokeAction(ControllerContext controllerContext, 
                                         string actionName)
        { ... }
    }
}
```

#### Built-In AsyncControllerActionInvoker

__Controller__ supports asynchronous operations by default, so __AsyncControllerActionInvoker__ is the current default implementation for `IActionInvoker`. This implementation is used by default when no custom invokers or invoker-factories are provided.

``` csharp
namespace System.Web.Mvc.Async
{
    public class AsyncControllerActionInvoker : ControllerActionInvoker, IAsyncActionInvoker
    {
        // ... other stuff
        
        public virtual IAsyncResult BeginInvokeAction(ControllerContext controllerContext, 
                                                      string actionName, 
                                                      AsyncCallback callback, 
                                                      object state)
        { ... }
        
        public virtual bool EndInvokeAction(IAsyncResult asyncResult) 
        { ... }
    }
}
```

#### Controller: The 'ActionInvoker' Property

Any provided `IActionInvoker` implementation becomes an infrastructure element of the __Controller__, accessible via the public __ActionInvoker__ property of it.

```csharp
namespace System.Web.Mvc
{
    public abstract class Controller : ControllerBase, ...
    {
        // ... omitted stuff
        
        public IActionInvoker ActionInvoker { get; set; } = CreateActionInvoker();
    }
}
```

#### Controller: The Process of ActionInvoker Instantiation

* Any provided implementation of `IActionInvoker` can be set via the public __ActionInvoker__ property.
* The `.CreateActionInvoker()` is the protected virtual method of the `Controller`, so we can easily override it when subclassing our __Controller__ to provide the specific process for action-invoker instantiation.
* The instantiation process makes use of two factory interfaces: `IActionInvokerFactory` and its async mate `IAsyncActionInvokerFactory`. Those factories can be customized in order to create an action invoker for each request. So we can provide our own factories implementations.
* For those cases when there are no factories provided, process makes a try to search for custom implementations of `IAsyncActionInvoker` and `IActionInvoker`. Async version is more appropriate.
* Ultimately, the default __AsyncControllerActionInvoker__ is used by the framework if no alternatives were provided.

``` csharp
namespace System.Web.Mvc
{
    public abstract class Controller : ControllerBase, ...
    {
        // ... omitted stuff
        
        protected virtual IActionInvoker CreateActionInvoker()
        {
            IAsyncActionInvokerFactory asyncActionInvokerFactory = 
                Resolver.GetService<IAsyncActionInvokerFactory>();
            
            if (asyncActionInvokerFactory != null)
            {
                return asyncActionInvokerFactory.CreateInstance();
            }
            
            IActionInvokerFactory actionInvokerFactory = 
                Resolver.GetService<IActionInvokerFactory>();
            
            if (actionInvokerFactory != null)
            {
                return actionInvokerFactory.CreateInstance();
            }
            
            return Resolver.GetService<IAsyncActionInvoker>()
                ?? Resolver.GetService<IActionInvoker>() 
                ?? new AsyncControllerActionInvoker();
        }
    }
}
```

#### Controller: Using ActionInvoker

As a recall, let's first see how the `IActionInvoker` is used by the __Controller__:

``` csharp
namespace System.Web.Mvc
{
    public abstract class Controller : ControllerBase, ...
    {
        // ... omitted stuff
        
        public IActionInvoker ActionInvoker { get { ... } set { ... } }
        
        protected override void ExecuteCore()
        {
            // ... omitted stuff
            
            if (!ActionInvoker.InvokeAction(ControllerContext, actionName))
            {
                HandleUnknownAction(actionName);
            }
            
            // ... omitted stuff
        }
    }
}
```

#### ControllerActionInvoker: The Process of .InvokeAction()

For the sake of brevity here we look through the sync version of action-invoke process of the __ControllerActionInvoker__:

``` csharp
public virtual bool InvokeAction(ControllerContext controllerContext, string actionName)
{
    ControllerDescriptor controllerDescriptor = GetControllerDescriptor(controllerContext);
    
    ActionDescriptor actionDescriptor = FindAction(controllerContext, 
                                                   controllerDescriptor,
                                                   actionName);

    FilterInfo filterInfo = GetFilters(controllerContext, 
                                       actionDescriptor);

    try
    {
        AuthenticationContext authenticationContext = 
            InvokeAuthenticationFilters(controllerContext, 
                                        filterInfo.AuthenticationFilters, 
                                        actionDescriptor);
        
        if (authenticationContext.Result != null)
        {
            /*
             * An authentication filter signalled that we should short-circuit the request.
             * Let all authentication filters contribute to an action result 
             * (to combine authentication challenges). Then, run this action result.
             */
             
            AuthenticationChallengeContext challengeContext = 
                InvokeAuthenticationFiltersChallenge(controllerContext, 
                                                     filterInfo.AuthenticationFilters, 
                                                     actionDescriptor,
                                                     authenticationContext.Result);

            InvokeActionResult(controllerContext, 
                               challengeContext.Result ?? authenticationContext.Result);
        }
        else
        {
            AuthorizationContext authorizationContext = 
                InvokeAuthorizationFilters(controllerContext, 
                                           filterInfo.AuthorizationFilters, 
                                           actionDescriptor);
            
            if (authorizationContext.Result != null)
            {
                /*
                 * An authorization filter signalled that we should short-circuit the request. 
                 * Let all authentication filters contribute to an action result 
                 * (to combine authentication challenges). Then, run this action result.
                 */
                 
                AuthenticationChallengeContext challengeContext = 
                    InvokeAuthenticationFiltersChallenge(controllerContext, 
                                                         filterInfo.AuthenticationFilters, 
                                                         actionDescriptor,
                                                         authorizationContext.Result);
                    
                InvokeActionResult(controllerContext, 
                                   challengeContext.Result ?? authorizationContext.Result);
            }
            else
            {
                if (controllerContext.Controller.ValidateRequest)
                {
                    ValidateRequest(controllerContext);
                }

                IDictionary<string, object> parameters = 
                                            GetParameterValues(controllerContext, 
                                                               actionDescriptor);

                ActionExecutedContext postActionContext = 
                                      InvokeActionMethodWithFilters(controllerContext, 
                                                                    filterInfo.ActionFilters, 
                                                                    actionDescriptor, 
                                                                    parameters);

                /*
                 * The action succeeded. Let all authentication filters contribute to an action result
                 * To combine authentication challenges; some authentication filters need to do negotiation
                 * even on a successful result). Then, run this action result.
                 */

                AuthenticationChallengeContext challengeContext = 
                    InvokeAuthenticationFiltersChallenge(controllerContext, 
                                                         filterInfo.AuthenticationFilters, 
                                                         actionDescriptor,
                                                         postActionContext.Result);
                    
                InvokeActionResultWithFilters(controllerContext, 
                                              filterInfo.ResultFilters,
                                              challengeContext.Result ?? postActionContext.Result);
            }
        }
    }
    catch (Exception ex)
    {
        // something blew up, so execute the exception filters
        ExceptionContext exceptionContext = 
                         InvokeExceptionFilters(controllerContext, 
                                                filterInfo.ExceptionFilters, 
                                                ex);

        if (!exceptionContext.ExceptionHandled)
        {
            throw;
        }
        
        InvokeActionResult(controllerContext, exceptionContext.Result);
    }

    return true;
}
```
