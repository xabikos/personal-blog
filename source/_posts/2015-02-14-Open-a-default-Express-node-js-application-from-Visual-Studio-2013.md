---
title: Open a default Express node.js application from Visual Studio 2013
date: 2015-02-14 13:30:00
tags: [Express, Visual Studio]
categories: [node.js]
alias: node.js/2015/02/14/open-a-default-express-node.js-application-from-visual-studio-2013.html
keywords:
- visual studio 2013
- node.js
- express
---

### node.js and Visual Studio

[node.js][node] is a very popular platform which allows having Javacript executed on the server. This is what everybody reads when navigating on the [official site][node]:
_Node.js® is a platform built on Chrome's JavaScript runtime for easily building fast, scalable network applications. Node.js uses an event-driven, non-blocking I/O model that makes it lightweight and efficient, perfect for data-intensive real-time applications that run across distributed devices._
<!-- more --> 
During the last couple of years Microsoft has put a lot of effort to provide a nice experience when we want to develop a node.js application by using Visual Studio. This comes of course in conjunction of supporting node.js hosting on the company's cloud platform [Microsoft Azure][nodeAzure]. We are able to develop any node.js application in Visual Studio because of the [Node.js Tools for Visual Studio][nodetools] awesome open source project. This is an extension that is completely free and anyone who is interested could install it on Visual Studio. After installing the extension we can create various node.js applications as we can see in the following picture.

{% image fancybox nocaption fig-100 clear group:vsnode newNodeProject.jpg "new node project" %}

#### Open an existing node application

The most interesting part according to my opinion is the first one, **From existing Node.js code,** as it gives us the opportunity to open an existing node application that was created from a different IDE and maybe on a different platform (Linux, OSX). This option gives us tremendous freedom to start contributing in a node.js project that some members of the team use different IDE and are on different platforms and we can still use Visual Studio without problem. Let me demonstrate how this works and also mention a fix to a minor problem.

#### Create a default Express application

The first step is to create the default Express application. I am not going to explain the prerequisites of doing this as [here][expressCreate] you can find very detailed instructions. After we have installed all the required dependencies and we have created a folder somewhere in our system, we should open a command window to the newly created folder and just hit:
{% tabbed_codeblock Create express application %}
    <!-- tab cmd -->
express firstExpressApplication
    <!-- endtab -->
{% endtabbed_codeblock %}
and after this is finished we have to move to the newly created folder and install the referenced node packages. This can be done my hitting:
{% tabbed_codeblock Move to the correct directory %}
    <!-- tab cmd -->
cd firstExpressApplication
npm install
    <!-- endtab -->
{% endtabbed_codeblock %}
At this point we have a very simple Express application which we can launch by typing in the command window
{% tabbed_codeblock Start express application %}
    <!-- tab cmd -->
set DEBUG=firstExpressApplication & node .\bin\www
    <!-- endtab -->
{% endtabbed_codeblock %}
After doing this just open a browser tab and navigate to the url: http://localhost:3000 and you should see the default express application running.

#### Open the Express application from Visual Studio

Now is the most interesting part where we can open this application from visual studio further change it. After giving a name in the window that is shown in the picture above we press next and we are presented with a window like the one following where we should choose the folder that contains the application's code.

{% image fancybox nocaption fig-100 clear group:vsnode selectforlder.jpg "select folder" %}

After clicking next a window like the following is shown to us where we can choose which file should be executed when F5 is pressed.

{% image fancybox nocaption fig-100 clear group:vsnode selectstartfile.jpg "select start file" %}

Here is the small problem as it doesn't give us the option to choose the bin\www file which contains the initialization logic. We have to manually change this after the Visual Studio project creation is finished. We are leaving the app.js file as selected and click next. The coming window is a great thought as it gives the flexibility of storing the Visual Studio project file in a different place than the original Express application. This means that we could even not add this file to source control and definitely it is not going to bother other people in the team that they don't use Visual Studio.

{% image fancybox nocaption fig-100 clear group:vsnode selectprojectlocation.jpg "select project location" %}

#### Fixing the problem of initial file

When the importing of the project is finished we can press F5 in order to achieve the same result as previously where we start the project from command line. But if we do this we will experience an error. A normal debugging session is starting and a command window with debugger details is shown but after a couple of seconds the window is disappeared and no application is present. This is due the starting file selection we made earlier. In order to fix this issue we need to edit the project and change the corresponding setting manually. We have to spot the **StartupFile** setting that is set in **app.js** and change it to **bin\www**. After saving the changes, reload the project and pressing F5 everything should work as expected.

You can find more details for node.js tools for visual studio in the [codeplex project][nodetools]

[node]: http://nodejs.org/
[nodeAzure]: http://azure.microsoft.com/en-us/develop/nodejs/
[nodetools]: https://nodejstools.codeplex.com/
[expressCreate]: http://expressjs.com/starter/generator.html