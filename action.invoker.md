## ActionInvoker

``` csharp
public interface IActionInvoker
{
    bool InvokeAction(ControllerContext controllerContext, 
                      string actionName
    );
}
```

===========================================================================
AsyncControllerActionInvoker : ControllerActionInvoker, IAsyncActionInvoker
===========================================================================

===============================================
ControllerActionInvoker : IActionInvoker
    InvokeAction
===============================================

public virtual bool InvokeAction(ControllerContext controllerContext, string actionName)
{
    ControllerDescriptor controllerDescriptor = GetControllerDescriptor(controllerContext);
    ActionDescriptor actionDescriptor = FindAction(controllerContext, controllerDescriptor, actionName);

    FilterInfo filterInfo = GetFilters(controllerContext, actionDescriptor);

    try
    {
        AuthenticationContext authenticationContext = InvokeAuthenticationFilters(controllerContext, filterInfo.AuthenticationFilters, actionDescriptor);

        if (authenticationContext.Result != null)
        {
            // An authentication filter signaled that we should short-circuit the request. Let all
            // authentication filters contribute to an action result (to combine authentication
            // challenges). Then, run this action result.
            AuthenticationChallengeContext challengeContext = InvokeAuthenticationFiltersChallenge(
                controllerContext, filterInfo.AuthenticationFilters, actionDescriptor,
                authenticationContext.Result);
            InvokeActionResult(controllerContext, challengeContext.Result ?? authenticationContext.Result);
        }
        else
        {
            AuthorizationContext authorizationContext = InvokeAuthorizationFilters(controllerContext, filterInfo.AuthorizationFilters, actionDescriptor);
            if (authorizationContext.Result != null)
            {
                // An authorization filter signaled that we should short-circuit the request. Let all
                // authentication filters contribute to an action result (to combine authentication
                // challenges). Then, run this action result.
                AuthenticationChallengeContext challengeContext = InvokeAuthenticationFiltersChallenge(
                    controllerContext, filterInfo.AuthenticationFilters, actionDescriptor,
                    authorizationContext.Result);
                InvokeActionResult(controllerContext, challengeContext.Result ?? authorizationContext.Result);
            }
            else
            {
                if (controllerContext.Controller.ValidateRequest)
                {
                    ValidateRequest(controllerContext);
                }

                IDictionary<string, object> parameters = GetParameterValues(controllerContext, actionDescriptor);
                ActionExecutedContext postActionContext = InvokeActionMethodWithFilters(controllerContext, filterInfo.ActionFilters, actionDescriptor, parameters);

                // The action succeeded. Let all authentication filters contribute to an action result (to
                // combine authentication challenges; some authentication filters need to do negotiation
                // even on a successful result). Then, run this action result.
                AuthenticationChallengeContext challengeContext = InvokeAuthenticationFiltersChallenge(
                    controllerContext, filterInfo.AuthenticationFilters, actionDescriptor,
                    postActionContext.Result);
                InvokeActionResultWithFilters(controllerContext, filterInfo.ResultFilters,
                    challengeContext.Result ?? postActionContext.Result);
            }
        }
    }
    catch (Exception ex)
    {
        // something blew up, so execute the exception filters
        ExceptionContext exceptionContext = InvokeExceptionFilters(controllerContext, filterInfo.ExceptionFilters, ex);
        if (!exceptionContext.ExceptionHandled)
        {
            throw;
        }
        InvokeActionResult(controllerContext, exceptionContext.Result);
    }

    return true;
}

============================================================
ControllerActionInvoker
    GetControllerDescriptor (abstract ControllerDescriptor)
============================================================

protected virtual ControllerDescriptor GetControllerDescriptor(ControllerContext controllerContext)
{
    // Frequently called, so ensure delegate is static
    Type controllerType = controllerContext.Controller.GetType();
    ControllerDescriptor controllerDescriptor = DescriptorCache.GetDescriptor(
        controllerType: controllerType,
        creator: (Type innerType) => new ReflectedControllerDescriptor(innerType),
        state: controllerType);
    return controllerDescriptor;
}

============================================================
ControllerActionInvoker
    FindAction (abstract ActionDescriptor)
============================================================

protected virtual ActionDescriptor FindAction(ControllerContext controllerContext, ControllerDescriptor controllerDescriptor, string actionName)
{
    if (controllerContext.RouteData.HasDirectRouteMatch())
    {
        List<DirectRouteCandidate> candidates = GetDirectRouteCandidates(controllerContext);

        DirectRouteCandidate bestCandidate = DirectRouteCandidate.SelectBestCandidate(candidates, controllerContext);
        if (bestCandidate == null)
        {
            return null;
        }
        else
        {
            // We need to stash the RouteData of the matched route into the context, so it can be
            // used for binding.
            controllerContext.RouteData = bestCandidate.RouteData;
            controllerContext.RequestContext.RouteData = bestCandidate.RouteData;

            // We need to remove any optional parameters that haven't gotten a value (See MvcHandler)
            bestCandidate.RouteData.Values.RemoveFromDictionary((entry) => entry.Value == UrlParameter.Optional);

            return bestCandidate.ActionDescriptor;
        }
    }
    else
    {
        ActionDescriptor actionDescriptor = controllerDescriptor.FindAction(controllerContext, actionName);
        return actionDescriptor;
    }
}