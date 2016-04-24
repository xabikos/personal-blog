---
title: ASP.NET - Create an OWIN middleware to trigger a permanent redirect
date: 2016-04-24 23:15:00
tags: [asp.net, owin, redirect]
categories: [web application, interception, middleware]
---

#### OWIN and project Katana

[OWIN][owin] (Open Web Iterface for .NET) is the new way of developing web applications in the .NET ecosystem. It is a great approach as it offers the flexibility of decoupling the application from the hosting environment itself. This means that as long as we develop a Web application on top of OWIN then we can easily host it outside the standard hosting environment so far which is IIS. A quick example is that by using a specific [nuget package] [selfhost]. As OWIN is just a specification then we need an implementation of this in each platform we want to support. The implementation for Microsoft's platform is called [Katana][katana].

#### OWIN core components

OWIN has a very nice and easy to understand architecture as it is closer to HTTP protocol and doesn't assume any functionality is there because of the hosting layer. All the manipulation has to do with the two main objects that is an HttpRequestMessage and HttpResponseMessage. This is a really good choice of abstracting the actual HTTP layer and it's an architecture that is very familiar to developers from other platforms like [node.js][node]. On top of this another very interesting and well known pattern that exists also in [Express][express] framework is the middlewares. A middleware is a really simple piece of code that gets a request object as a parameter and can either transform this request and pass it to the next middleware if present or create a response and don't allow the request to further execute on the server. So a web application could potentially consist of several middlewares that are executed one after the other.

#### OWIN middleware details

The bare minimum OWIN middleware could look like the code below which the only thing it does is to write some statements as diagnostic and lets the request to proceed by calling the next middleware in the pipeline. 

{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
public class MinimumMiddleware
{
    private AppFunc _next;

    public void Initialize(AppFunc next)
    {
        _next = next;
    }

    public async Task Invoke(IDictionary<string, object> environment)
    {
        Debug.WriteLine("Begin executing request in custom middleware");
        await next.Invoke(environment);
        Debug.WriteLine("End executing request in custom middleware");
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

In order to use this middleware we have to register it in the StartUp class as below.

{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
public partial class Startup {
    public void Configuration(IAppBuilder app) {
        var customMiddleware = new MinimumMiddleware();
        app.Use(customMiddleware);
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

#### Trigger a redirect from a middleware

The implementation above gives us just a dictionary of string and object and into this dictionary exists all the useful info like the request itself. A better option is to derive from OwinMiddleware class that gives us access to all the request specific data in a strongly type manner through IOwinContext interface. In the code that follows we can see an example of triggering a permanent redirect from an OWIN middleware and don't let the request to further processed by the application.

{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
public class CheckUrlMiddleware : OwinMiddleware
{
    private readonly IUrlChecker _checker;

    public CheckLegacyUrlMiddleware(OwinMiddleware next, IUrlChecker checker) : base(next)
    {
        _checker = checker;
    }

    public async override Task Invoke(IOwinContext context)
    {
        
        var url = context.Request.Uri;
        string urlToRedirect = "";        
        if (_checker.NeedsPermanentRedirect(url.AbsoluteUri, out urlToRedirect) )
        {
            // 301 is the status code of permanent redirect
            context.Response.StatusCode = 301;
            context.Response.Headers.Set("Location", urlToRedirect);
        }
        else
        {
            await Next.Invoke(context);
        }
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

The IUrlChecker is just an Interface that the implementation of it contains all the business logic if a redirect should take place or not. I use it here also to demonstrate how we can inject dependencies in our middleware in the correct way. The piece of code below shows what we have to do in order this middleware to executed for every request.

{% tabbed_codeblock UserConfirmedFilterAttribute %}
    <!-- tab cs -->
public partial class Startup {
    public void Configuration(IAppBuilder app) {
        var urlChecker = DependencyResolver.Current.GetService<IUrlChecker>();
        app.Use<CheckUrlMiddleware>(urlChecker);        
    }
}
    <!-- endtab -->
{% endtabbed_codeblock %}

[owin]: http://owin.org/
[selfhost]: https://www.nuget.org/packages/Microsoft.Owin.SelfHost/
[katana]: http://www.asp.net/aspnet/overview/owin-and-katana/an-overview-of-project-katana
[node]: http://nodejs.org/
[express]: http://expressjs.com/