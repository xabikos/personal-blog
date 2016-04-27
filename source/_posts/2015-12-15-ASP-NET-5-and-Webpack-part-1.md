---
title: ASP.NET 5 and Webpack part 1
date: 2015-12-15 18:40:00
tags: [asp.net 5, webpack]
categories: [javascript module bundler, javascript dependencies management, css module bundler, css dependencies management]
alias: javascript module bundler/javascript dependencies management/css module bundler/css dependencies management/2015/12/15/asp.net-5-and-webpack-part-1.html
keywords:
- webpack
- asp.net core
- asp.net vnext
---

### Introduction
This is the first of the two parts about using [webpack][webpack] and the corresponding [nuget package][nuget] I created, in an asp.net 5 application.
This part acts as an introduction to the reasons behind the need for an alternative approach of static asset management and JavaScript module bundling in an asp.net 5 application.
Besides that includes a brief explanation of an example Reactjs application that uses the webpack nuget package. In the second part we will explore the internals of the nuget package itself and dig into webpack's details.
<!-- more -->
### Connection to the past
A long time ago I wrote a [post][previous-post] about using [webpack][webpack]
in an asp.net 5 (vnext at that point) application. I ended that post saying that another post would come with a full isomorphic JavaScript application in asp.net with [React.js][react] and webpack.
Unfortunatelly I didn't keep that promise but I am very optimistic this will happen in the future especially after we will be able to use the amazing [Edge.js][edgejs] library for server side rendering of React.js components.
In the meantime I was involved in several React projects and in all of them was using webpack so I had the change to really dig into the tool and learn it deeply.

### Why to use webpack
The last personal side project I am working on is a Reactjs application with asp.net 5 as server side platform this time. I really like the new approach from asp.net platform and the flexibility it provides.
In the tooling section though, there is still plenty of room for improvement. As mentioned above I was involved in several Reactjs projects and none of them was using asp.net as back end. I learned a lot during that time
but I was also a bit frustrated about the lack of tooling in .net ecosystem. As I highlighted in my [related blog post][previous-post] the new template provides a new and better way for client side dependencies maangement
through tools like [bower][bower], [gulp][gulp] and [grunt][grunt]. My sense though is that community walks away from this approach and there is a tendency to use just npm modules even for client side packages and not [bower][bower].
On top of that more and more projects use npm scripts and webpack for executing tasks instead of [gulp][gulp] or [grunt][grunt] something I really like as it removes a dependency and a corresponding configuration file from the project.
Finally I really suggest to watch [Jonathan Creamer][jonathan] [talk][talk] on some advanced features of webpack.

### Which problem the library solves
One of the most valuable features of webpack is the accompanied [development server][devserver] and the live reload it provides when saving a file.
Of course this is not something new as there are tools that doing this already but where webpack-dev-server shines is that does this without reloading the page.
This happens by default for stylesheets. When using Reactjs we have exactly the same behaviour by using the appropriate configuration something that is extremelly powerfull as we dont's loose the application's state from a full page refresh.

What it was frustrated and lead me to create that library is the fact that I had first to start webpack-dev-server and then start my project through Visual Studio or command line in a different platform.
On top of that I had to do all the webpack configuration and add the correct script tags in my cshtml layout files. The webpack-dev-server could serve static html files as well and inject the approperiate scripts automatically
but then you end up to have your appication running in a diffrent location than the back end during development and have to solve all the [CORS][cors] issues.
So during all these cycles I realised I should automate all these steps, from basic configuration to launch the right tool every time I am starting my asp.net project. The configuration the [nuget package][nuget] provides looks like:

{% tabbed_codeblock Start asp.net core application %}
    <!-- tab cs -->
app.UseWebpack(new WebpackOptions() {
    StylesTypes = new List<StylesType>() {
        StylesType.Css
    },
    EnableHotLoading = true,
});
    <!-- endtab -->
{% endtabbed_codeblock %}

### A concrete example
In order to demonstrate webpack in an asp.net 5 in a better way I have created a boilerplate application in [Github][project].
Once you copy the repository you need to execute dnu restore and dnu build as usual from the command line to download all the nuget packages and after this npm install to install the required node packages that are a few.
After finishing that, you are ready to start the application by executing dnx web. This will start the normal asp.net server but the webpack-dev-server too in the address that is provided in the configuration.
If no configuration is present will start on localhost:4000. The webpack-dev-server logs to the same console as asp.net does and you can get a nice indication on what is happening.
After the initial launch and when everything is fine you should have in the console an output like the following:

{% tabbed_codeblock Webpack console output %}
    <!-- tab cmd -->
[279] ./~/react-deep-force-update/lib/index.js 1.29 kB {0} [built]
[280] ./~/global/window.js 243 bytes {0} [built]
[281] ./app/components/counter.jsx 3.63 kB {0} [built]
webpack: bundle is now VALID.
    <!-- endtab -->
{% endtabbed_codeblock %}

And if you browse to the project (http://localhost:5000 by default) you should get a screen like the one that follows:

{% image fancybox nocaption fig-100 clear group:webpack webpackBoilerplate.png "webpack" %}

The output is quite simple but it demonstrates most of the features that will be used in a real application as well. It contains a statefull component that has some basic styling through sass.
The component actually has two counters. The first just counts the seconds since it was loaded in the page and the second the number of rerenders. Initially is maybe hard to understand the difference but once you will start
changing the component you can see that the rerender counter will be increased more than time elapsed as any chnage in the component will trigger an extra render.
**The amazing thing is that we can edit both the styles and the output of the component without loosing it's state**

{% image fancybox nocaption fig-100 clear group:webpack hotloading.gif "webpack" %}

### Additional notes
- Used only during development

   This is a library that should be used only during development environment and should not be part of any production system.
   To be sure you are staring in a development environment from the command line you need to execute `dnx web ASPNET_ENV=Development`.

- Cross platform solution for live reload of static resources in asp.net 5 applications

   As mentioned before there are some other tools that provide live reload functionality. This is a well tested solution for an asp.net 5 application that works cross platform.
   Actually I developed the boilerplate project in OSX by using [Visual Studio Code][code]

- Production bundles

   Webpack is a great tool for preparing the production bundles too as it can handle minification, create separate bundle for vendor scripts and support async loading of the bundles.
   The tool has many more capabilities and would require a separate blog post to describe it.


[webpack]: http://webpack.github.io/
[nuget]: https://www.nuget.org/packages/Webpack/
[previous-post]: http://xabikos.com/javascript%20module%20bundler/javascript%20dependencies%20management/css%20module%20bundler/css%20dependencies%20management/2015/05/17/asp.net-vnext-with-webpack.html
[react]: http://facebook.github.io/react/
[edgejs]: http://tjanczuk.github.io/edge/#/
[bower]: http://bower.io
[gulp]: http://gulpjs.com
[grunt]: http://gruntjs.com
[devserver]: https://webpack.github.io/docs/webpack-dev-server.html
[cors]: https://en.wikipedia.org/wiki/Cross-origin_resource_sharing
[jonathan]: https://twitter.com/jcreamer898
[talk]: https://www.youtube.com/watch?v=MzVFrIAwwS8
[project]: https://github.com/xabikos/aspnet5-react-webpack-boilerplate
[code]: https://code.visualstudio.com