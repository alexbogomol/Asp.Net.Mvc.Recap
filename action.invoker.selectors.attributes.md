### Customizing the Process of Action and Method Selection

__ControllerActionInvoker__ has a conventions-driven process of action-decision. This process can be easily customized to our needs with the influenсe of attributes, annotating the decided action methods of the controller.

This effect can be achieved by deriving from the abstractions, used by __ControllerActionInvoker__:
* __ActionNameSelectorAttribute__
* __ActionMethodSelectorAttribute__

#### Process of Action Name Selection

Firstly, __ControllerActionInvoker__ decides the name of the action to be executed. This step can be influenсed with help of attributes, deriving from the abstract __ActionNameSelectorAttribute__:
* Built-In __ActionNameAttribute__
* Custom implementations

##### Action Name Selector Abstraction

``` csharp
using System.Reflection;

namespace System.Web.Mvc
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public abstract class ActionNameSelectorAttribute : Attribute
    {
        public abstract bool IsValidName(ControllerContext controllerContext, 
                                         string actionName, 
                                         MethodInfo methodInfo);
    }
}
```

##### Built-In Implementation for Action Name Selector

```csharp
using System.Reflection;
using System.Web.Mvc.Properties;

namespace System.Web.Mvc
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public sealed class ActionNameAttribute : ActionNameSelectorAttribute
    {
        public ActionNameAttribute(string name)
        {
            if (String.IsNullOrEmpty(name))
            {
                throw new ArgumentException(MvcResources.Common_NullOrEmpty, "name");
            }

            Name = name;
        }

        public string Name { get; private set; }

        public override bool IsValidName(ControllerContext controllerContext, 
                                         string actionName, 
                                         MethodInfo methodInfo)
        {
            return String.Equals(actionName, Name, StringComparison.OrdinalIgnoreCase);
        }
    }
}
```

#### Process of Action Method Selection

When __ControllerActionInvoker__ knows the name of the action to be executed, it must decide the corresponding controller method for this action name.

We can disambiguate the decision-process with help of attributes, deriving from the abstract __ActionMethodSelectorAttribute__:
* Built-In __Http[Verb]Attribute__
* Built-In __NonActionAttribute__
* Custom Implementations

##### Action Method Selector Abstraction

```csharp
using System.Reflection;

namespace System.Web.Mvc
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public abstract class ActionMethodSelectorAttribute : Attribute
    {
        public abstract bool IsValidForRequest(ControllerContext controllerContext,
                                               MethodInfo methodInfo);
    }
}
```

##### Custom Action Method Selector Implemetation

All we have to do when defining our custom __ActionMethodSelectorAttribute__ - is just answer the question:
* _"Is the annotated method valid for this request?"_

``` csharp
public class LocalAttribute : ActionMethodSelectorAttribute
{
    public override bool IsValidForRequest(ControllerContext controllerContext,
                                           MethodInfo methodInfo)
    {
        return controllerContext.HttpContext.Request.IsLocal;
    }
}
```

##### Some Built-In Implementations for Action Method Selectors

``` csharp
using System.Reflection;

namespace System.Web.Mvc
{
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public sealed class NonActionAttribute : ActionMethodSelectorAttribute
    {
        public override bool IsValidForRequest(ControllerContext controllerContext, 
                                               MethodInfo methodInfo)
        {
            return false;
        }
    }
    
    [AttributeUsage(AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
    public sealed class HttpPostAttribute : ActionMethodSelectorAttribute
    {
        private static readonly AcceptVerbsAttribute _innerAttribute = 
            new AcceptVerbsAttribute(HttpVerbs.Post);

        public override bool IsValidForRequest(ControllerContext controllerContext, 
                                               MethodInfo methodInfo)
        {
            return _innerAttribute.IsValidForRequest(controllerContext, methodInfo);
        }
    }
}
```

#### Using the attributes

```csharp
public class CustomerController : Controller 
{
    // GET: /Customer/Index
    
    public ViewResult Index()
    {
        return View("Result", new Result{ ... });
    }
    
    // GET: /Customer/Enumerate
    
    [ActionName("Enumerate")]
    public ViewResult List()
    {
        return View("Result", new Result { ... });
    }
    
    // POST: /Customer/Save
    
    [ActionName("Save")]
    [HttpPost]
    public RedirectResult Store(Customer customer)
    {
        Db.Customers.Add(customer);
        
        return RedirectToAction("Index");
    }

    [NonAction]
    public ActionResult MyAction()
    {
        return View();
    }
}
```
