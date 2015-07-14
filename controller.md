
public interface IController
{
    void Execute(RequestContext requestContext);
}

public interface IControllerFactory
{
    IController CreateController(RequestContext requestContext, string controllerName);
    SessionStateBehavior GetControllerSessionBehavior(RequestContext requestContext, string controllerName);
    void ReleaseController(IController controller);
}

==================================================
ControllerBase : IController
    Execute
    Initialize
    ExecuteCore
==================================================

public ControllerContext ControllerContext { get; set; }

protected virtual void Execute(RequestContext requestContext)
{
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

==================================================
Controller : ControllerBase
    ExecuteCore
    ActionInvoker : IActionInvoker
==================================================

protected override void ExecuteCore()
{
    // If code in this method needs to be updated, please also check the BeginExecuteCore() and
    // EndExecuteCore() methods of AsyncController to see if that code also must be updated.

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

public IActionInvoker ActionInvoker
{
    get { return _actionInvoker ?? (_actionInvoker = CreateActionInvoker()); }
    set { _actionInvoker = value; }
}

protected virtual IActionInvoker CreateActionInvoker()
{
    // Controller supports asynchronous operations by default. 
    // Those factories can be customized in order to create an action invoker for each request.
    var asyncActionInvokerFactory = Resolver.GetService<IAsyncActionInvokerFactory>();
    if (asyncActionInvokerFactory != null)
    {
        return asyncActionInvokerFactory.CreateInstance();
    }
    
    var actionInvokerFactory = Resolver.GetService<IActionInvokerFactory>();
    if (actionInvokerFactory != null)
    {
        return actionInvokerFactory.CreateInstance();
    }

    // Note that getting a service from the current cache will return the same instance for every request.
    return Resolver.GetService<IAsyncActionInvoker>()
        ?? Resolver.GetService<IActionInvoker>() 
        ?? new AsyncControllerActionInvoker();
}
