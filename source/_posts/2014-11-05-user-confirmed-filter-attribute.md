---
title: "ASP.NET User confirmed email filter attribute"
date: 2014-11-05 20:00:00
tags: [asp.net mvc, asp.net web api, action filters]
categories:	[security, identity system]
alias: security/identity system/2014/11/05/user-confirmed-filter-attribute.html
keywords:
- asp.net
- identity
- security
---

[Action filters][af] is a very powerful mechanism in an ASP.NET MVC application that gives us the capability of injecting functionality in our application in a centralized and structure way. 
In this post I am showing how we can create an action filter attribute in order to decorate our controllers to check if the current user has verified his email address 
after registering in our system through new ASP.NET [Identity system][identity]. I am going to create two filters as we want to have this functionality in regular MVC controllers and in Web Api controllers as well.

First lets see the code for the regular MVC controllers. What we have to do is just derive from ActionFilterAttribute class that lives in System.Web.Mvc namespace and override the OnActionExecuting method.
{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true, Inherited = true)]
public class UserConfirmedFilterAttribute : ActionFilterAttribute {
    public override void OnActionExecuting(ActionExecutingContext filterContext) {
        var userId = filterContext.HttpContext.User.Identity.GetUserId();
        // User is not logged in so redirect him to log in controller action
        if (string.IsNullOrEmpty(userId)) {
            filterContext.Result = new RedirectToRouteResult(new RouteValueDictionary(
                        new { controller = "Account", action = "Login", 
                                returnUrl = filterContext.HttpContext.Request.RawUrl }));
            return;
        }

        var userManager = filterContext.HttpContext.GetOwinContext().GetUserManager<ApplicationUserManager>();
        if (!userManager.IsEmailConfirmed(userId)) {
            filterContext.Result =
                new RedirectToRouteResult(
                    new RouteValueDictionary(new { controller = "ConstrollerNameToRedirect", 
                                    action = "ActionMethodToRedirect" }));
            return;
        }
        base.OnActionExecuting(filterContext);
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

The code is very simple as in the case the user has confirmed the email the execution flow continues normally. In the opposite case we are able to redirect the user 
in a specific action of a controller by replacing the corresponding values.

The code is very similar for Web Api controllers as we can see right below.

{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = true, Inherited = true)]
public class UserConfirmedWebApiFilterAttribute : ActionFilterAttribute {
    public override Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken) {
        var userId = actionContext.RequestContext.Principal.Identity.GetUserId();
        // User is not logged in so redirect him to log in controller action
        if (string.IsNullOrEmpty(userId)) {
            actionContext.Response = actionContext.Request.CreateErrorResponse(HttpStatusCode.Unauthorized,
                "You must be logged in to access this resource");
            return Task.FromResult(0);
        }

        var userManager = actionContext.Request.GetOwinContext().GetUserManager<ApplicationUserManager>();
        if (!userManager.IsEmailConfirmed(userId)) {
            actionContext.Response = actionContext.Request.CreateErrorResponse(HttpStatusCode.BadRequest,
                "You must be verify your email address in order to access this resource");
            return Task.FromResult(0);
        }
        return base.OnActionExecutingAsync(actionContext, cancellationToken);
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

Here I choose to implement the async version of OnActionExecuting method. If the user has confirmed the email address then the request is served normally. 
If the user is not logged in or has not confirmed the email a bad request response is returned containing the additional message.

You can find a complete [project][project] containing both of the attributes on Github.
	
[af]: https://www.asp.net/mvc/overview/older-versions-1/controllers-and-routing/understanding-action-filters-cs/
[identity]: http://www.asp.net/identity
[project]: https://github.com/xabikos/ActionFilters
