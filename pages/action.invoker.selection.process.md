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

#### ControlerDescriptor Abstraction

``` csharp
namespace System.Web.Mvc
{
    public abstract class ControllerDescriptor : ICustomAttributeProvider, IUniquelyIdentifiable
    {
        public virtual string ControllerName { get { ... } }

        public abstract Type ControllerType { get; }

        public abstract ActionDescriptor FindAction(ControllerContext controllerContext, 
                                                    string actionName);

        public abstract ActionDescriptor[] GetCanonicalActions();

        public virtual IEnumerable<FilterAttribute> GetFilterAttributes(bool useCache) { ... }

		// ICustomAttributeProvider Implementation
        public virtual object[] GetCustomAttributes(bool inherit) { ... }

        public virtual object[] GetCustomAttributes(Type attributeType, bool inherit) { ... }

        public virtual bool IsDefined(Type attributeType, bool inherit) { return false; }

		// IUniquelyIdentifiable Implementation
        public virtual string UniqueId { get { ... } }
    }
}
```

#### ActionDescriptor Method Verification

Action-method candidate must pass through the following set of rules to be considered as _callable_:
* it cannot be __static__
* it cannot be a method of __non-ControllerBase__-inherited instance
* it cannot be __generic__ method
* it cannot be a method with __ref/out__ parameters

``` csharp
namespace System.Web.Mvc
{
    public abstract class ActionDescriptor : ICustomAttributeProvider, IUniquelyIdentifiable
    {
        // ... omitted stuff
        
        internal static string VerifyActionMethodIsCallable(MethodInfo methodInfo)
        {
            // we can't call static methods
            if (methodInfo.IsStatic)
            {
                return "Cannot Call Static Method";
            }

            // we can't call instance methods where the 'this' parameter 
            // is a type other than ControllerBase
            if (!typeof(ControllerBase).IsAssignableFrom(methodInfo.ReflectedType))
            {
                return "Cannot Call Instance Method On Non-Controller Type";
            }

            // we can't call methods with open generic type parameters
            if (methodInfo.ContainsGenericParameters)
            {
                return "Cannot Call Open Generic Methods";
            }

            // we can't call methods with ref/out parameters
            ParameterInfo[] parameterInfos = methodInfo.GetParameters();
            foreach (ParameterInfo parameterInfo in parameterInfos)
            {
                if (parameterInfo.IsOut || parameterInfo.ParameterType.IsByRef)
                {
                    return "Cannot Call Methods With Out Or Ref Parameters";
                }
            }

            // we can call this method
            return null;
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
