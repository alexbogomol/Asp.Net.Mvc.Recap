### Process of Action and Method Selection

As a recall, let's see how the __ControllerActionInvoker__ describes the current controller and discovers action methods, available to be called.

``` csharp
namespace System.Web.Mvc
{
    public class ControllerActionInvoker : IActionInvoker
    {
        // ... omitted stuff
        
        public virtual bool InvokeAction(ControllerContext controllerContext, string actionName)
        {
            ControllerDescriptor controllerDescriptor = GetControllerDescriptor(controllerContext);
            
            ActionDescriptor actionDescriptor = FindAction(controllerContext, 
                                                           controllerDescriptor,
                                                           actionName);
        
            FilterInfo filterInfo = GetFilters(controllerContext, 
                                               actionDescriptor);
            
            // ... omitted stuff
        }
    }
}
```

#### Get the ControllerDescriptor for ControllerActionInvoker

__ControllerDescriptor__ abstraction is used by the __ControllerActionInvoker__ to get the listing of actions, available to call. The default implementation for __ControllerDescriptor__ is __ReflectedControllerDescriptor__.

``` csharp
protected virtual ControllerDescriptor GetControllerDescriptor(ControllerContext controllerContext)
{
    // Frequently called, so ensure delegate is static
    Type controllerType = controllerContext.Controller.GetType();

    ControllerDescriptor controllerDescriptor = 
                         DescriptorCache.GetDescriptor(
                             controllerType: controllerType,
                             creator: (Type innerType) => new ReflectedControllerDescriptor(innerType),
                             state: controllerType);

    return controllerDescriptor;
}

```

#### FindAction process for ControllerActionInvoker

``` csharp
protected virtual ActionDescriptor FindAction(ControllerContext controllerContext, 
                                              ControllerDescriptor controllerDescriptor, 
                                              string actionName)
{
    if (controllerContext.RouteData.HasDirectRouteMatch())
    {
        List<DirectRouteCandidate> candidates = GetDirectRouteCandidates(controllerContext);

        DirectRouteCandidate bestCandidate = 
                             DirectRouteCandidate.SelectBestCandidate(candidates, controllerContext);
        
        if (bestCandidate == null)
        {
            return null;
        }
        else
        {
            // We need to stash the RouteData of the matched route 
            // into the context, so it can be used for binding.
            controllerContext.RouteData = bestCandidate.RouteData;
            controllerContext.RequestContext.RouteData = bestCandidate.RouteData;

            // We need to remove any optional parameters 
            // that haven't gotten a value (See MvcHandler)
            bestCandidate.RouteData.Values.RemoveFromDictionary(
                (entry) => entry.Value == UrlParameter.Optional);

            return bestCandidate.ActionDescriptor;
        }
    }
    else
    {
        ActionDescriptor actionDescriptor = 
                         controllerDescriptor.FindAction(controllerContext, actionName);
                         
        return actionDescriptor;
    }
}
```

##### References:
* http://www.pieterg.com/2013/4/aspnet-mvc-under-the-hood-part-5
* http://www.developermemo.com/3068327/
