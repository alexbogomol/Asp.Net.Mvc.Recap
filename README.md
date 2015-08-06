### Asp.Net Mvc Recap

My notes on internals of Asp.Net Mvc framework

#### Mvc LifeCycle in terms of abstractions
 
All construction components of Asp.Net Mvc framework are coupled via abstractions, predominantly interfaces.
 
Step | Abstraction                                | Built-In Implementation
---  | ---                                        | ---
1    | `IRouteHandler.GetHttpHandler`             | [__MvcRouteHandler__](/pages/mvc.routing.md)
2    | `IHttpHandler.ProcessRequest`              | [__MvcHandler__](/pages/mvc.routing.md)
3    | [`IControllerFactory.CreateController`](/pages/controller.factory.abstraction.md) | [__DefaultControllerFactory__](/pages/controller.factory.default.md)
4    | [`IControllerActivator.Create`](/pages/controller.activator.abstraction.md) | __DefaultControllerActivator__
5    | `IController.Execute`                      | [__Controller : ControllerBase__](/pages/controller.md)
6    | [`IActionInvoker.InvokeAction`](/pages/action.invoker.abstraction.md) | [__AsyncControllerActionInvoker__](/pages/action.invoker.default.md)
7    | `IAuthenticationFilter[].OnAuthentication` | 
8    | `IAuthorizationFilter[].OnAuthorization`   | __AuthorizeAttribute__
9    | `IActionFilter[].OnActionExecuting`        | __ActionFilterAttribute__
10   | `IActionFilter[].OnActionExecuted`         | __ActionFilterAttribute__
11   | `IResultFilter[].OnActionExecuting`        | __ActionFilterAttribute__
12   | `IResultFilter[].OnActionExecuted`         | __ActionFilterAttribute__
...  | ...                                        | ...
