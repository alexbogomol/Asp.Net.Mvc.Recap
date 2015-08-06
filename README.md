### Asp.Net Mvc Recap

My notes on internals of Asp.Net Mvc framework

#### Mvc LifeCycle in terms of abstractions
 
All construction components of Asp.Net Mvc framework are coupled via abstractions, predominantly interfaces.
 
Step | Abstraction                                | Implementation
---  | ---                                        | ---
1    | `IRouteHandler.GetHttpHandler`             | __MvcRouteHandler__
2    | `IHttpHandler.ProcessRequest`              | __MvcHandler__
3    | `IControllerFactory.CreateController`      | __DefaultControllerFactory__
4    | `IControllerActivator.Create`              | __DefaultControllerActivator__
5    | `IController.Execute`                      | __Controller : ControllerBase__
6    | [`IActionInvoker.InvokeAction`](/pages/action.invoker.abstraction.md) | [__AsyncControllerActionInvoker__](/pages/action.invoker.default.md)
7    | `IAuthenticationFilter[].OnAuthentication` | 
8    | `IAuthorizationFilter[].OnAuthorization`   | __AuthorizeAttribute__
9    | `IActionFilter[].OnActionExecuting`        | __ActionFilterAttribute__
10   | `IActionFilter[].OnActionExecuted`         | __ActionFilterAttribute__
11   | `IResultFilter[].OnActionExecuting`        | __ActionFilterAttribute__
12   | `IResultFilter[].OnActionExecuted`         | __ActionFilterAttribute__
...  | ...                                        | ...
