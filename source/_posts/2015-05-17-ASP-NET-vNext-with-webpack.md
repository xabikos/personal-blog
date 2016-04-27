---
title: ASP.NET vNext with webpack
date: 2015-05-17 20:10:00
tags: [asp.net vnext, webpack]
categories: [javascript module bundler, javascript dependencies management, css module bundler, css dependencies management]
alias: javascript module bundler/javascript dependencies management/css module bundler/css dependencies management/2015/05/17/asp.net-vnext-with-webpack.html
keywords:
- webpack
- asp.net core
- asp.net vnext
---

#### Why I was absent

It is a long time from my last post but so much have happened during this time period. Initially I was involved in the organization of local [Global Azure Bootcamp][bootcamp] in [Thessaloniki][skgazure]. It was our first attempt to organize an one day event and we are very satisfied by the result. It is an amazing feeling to be part of a global community for one day. Right after Azure Bootcamp, Microsoft's biggest development conference took place in San Fransisco. At [Build][build] tons of new features were announced and I am still straggling to watch all the interesting videos on [Channel 9][channel9]. Lastly I attended the first international development conference that was organized in the city I am living, in Thessaloniki. [Devit][devit] took place on Friday 15th of May and it was very successful and amazingly well organized conference.

#### ASP.NET vNext

[asp.net vNext][vnext] is the next version of the well know web stack framework and I won't describe it at all as it's quite easy to find details about it. It's not yet production ready but that point comes really close. I am very interested on this new web stack and the direction it gets so I am playing with it when the time permits. Among the other changes the core team has done, they also changed the way the client side package management is done. The framework adopts well known practices on the field and adds to the core system tools like [npm][npm] and [bower][bower] package managers and [gulp][gulp] and [grunt][grunt] tasks runners. For the people that are not familiar with basic [nodejs][node] development and client side package management they should spend some time and study those as it's part of the next version of asp.net. An excellent place to start the reading is the newly added official [documentation site][aspdoc] of asp.net vNext.
<!-- more -->
#### Client side package management and webpack

During the last months in my work I am heavily involved on developing a large [Reactjs][react] application. For this reason we need to use a lot of node and bower packages something that makes the usage of a module bundler an absolute requirement. After some investigation we chose to use [webpack][webpack] as it is very popular, it has an active community behind it and it fits perfect with react development. One of the greatest features of webpack is that it allows to use node packages directly inside the browser.

#### Combine ASP.NET vNext with webpack

 The current project templates for asp.net vNext that are part of Visual Studio 2015 RC offer a completely different way of managing the client side packages dependencies which is in the right direction. I will try to make a step further and show how we can use webpack to organize our JavaScript and css code in a more modular way. To demonstrate this I have created [a repository in Github][repo] which would be a good starting point. After cloning the repository locally there are a sequence of commands you should execute in order to launch the application. Before executing these commands you have to make sure that [nodejs][node] is properly install on your machine and also the latest asp.net vNext is also present. You can get more info on how to install the asp.net vNext again in the [official documentation site][aspdoc]. Once the environment is ready you need to execute form a command line that points to the same folder you cloned the repository the command:

 {% tabbed_codeblock Install node modules %}
    <!-- tab cmd -->
 	npm install
     <!-- endtab -->
{% endtabbed_codeblock %}

 in order to install all the node packages. Then we need to install all the referenced bower components by executing first

{% tabbed_codeblock Install bower globally %}
    <!-- tab cmd -->
 	npm install bower -g
     <!-- endtab -->
{% endtabbed_codeblock %}

 to install Bower globally and then

 {% tabbed_codeblock Install bower modules view %}
    <!-- tab cmd -->
 	bower install
     <!-- endtab -->
{% endtabbed_codeblock %}

 to install the bower components. After doing this you need to generate the bundles through the preconfigured gulp task. So you need to execute

{% tabbed_codeblock Install gulp globally %}
    <!-- tab cs -->
 	npm install gulp -g
     <!-- endtab -->
{% endtabbed_codeblock %}

 to install Gulp as a global package and then

{% tabbed_codeblock Start gulp development task %}
    <!-- tab cs -->
 gulp development
    <!-- endtab -->
{% endtabbed_codeblock %}

in order to create the JavaScript and css bundles. The final step is to launch the development server by executing

{% tabbed_codeblock Start asp.net core application %}
    <!-- tab cs -->
 dnx . web
    <!-- endtab -->
{% endtabbed_codeblock %}

After a while you should see Started on the command line and if you navigate to [http://localhost:5000/][local] you should get a page like the one following

{% image fancybox nocaption fig-100 clear group:webpack webpack.png "webpack" %}

#### JavaScript dependencies management

The application is really basic and what it does is after entering a number in the textbox and pressing submit renders some Lorem ipsum text on the screen. The code for this exists in the index.cshtml file is pretty basic and easy to understand. But the interesting is how the Lorem global variable is available. This is done because in the _Layout.cshtml we include the app.bunle.js file which was generated by webpack when we executed the gulp development command. If you try to explore this file you would probably feel a bit confused as it contains many lines of code. But let's take one step back and see where this file came from.

Let's start by looking at the file lorem.js which the content is following

{% tabbed_codeblock A simple javascript module %}
    <!-- tab js -->
import loremHipsum from 'lorem-hipsum';

let Lorem = {
  addText(count, loremTextElement, loremValidationElement) {
	if (isNaN(count) || count < 1 || count > 100) {
	  loremValidationElement.html('Invalid input');
	} else {
	  loremTextElement.html(loremHipsum({
        count: count,
	    units: 'sentences'
	  }));
    }
  }
};

export default Lorem;
    <!-- endtab -->
{% endtabbed_codeblock %}

As you can see we can use the new ES6 module system thanks to webpack and the [babel loader][babelloader]. The lorem-hipsum is just on ordinal node package that we can consume directly inside our browser scripts. Also we export the object Lorem we created in the file. The object just contains a simple method that uses the node package to add some random text to the provided element. You should also notice that we can use the new way of declaring functions.

The next important piece in the procedure is the app.js file which contains only two lines of code.

{% tabbed_codeblock Client side app entry point %}
    <!-- tab js -->
var Lorem = require('expose?Lorem!./lorem')
require('../styles/site.scss');
    <!-- endtab -->
{% endtabbed_codeblock %}

This is the entry point in our application and the first line is the one that makes the Lorem variable available in the global scope through the [expose-loader][expose] webpack loader. The second line is related to css and we will explain it afterwards. So during the creation of the bundle webpack solves all the dependencies starting from app.js file and produces the final app.bundle.js file which can be referenced directly from an html page.

#### css dependencies management

Let's move on how webpack can assist us on stylesheets dependencies management. If everything is fine so far when you load the page you will notice that the top header animates and fades in to the page. This is done by the [animated.sass][animated] bower component which is referenced and applying certain classes. The most interesting piece of this exists in the file main.scss which looks like this:

{% tabbed_codeblock Application's style file %}
    <!-- tab css -->
@import "../bower_components/animated.sass/_animated.scss";
@include animation(fade-in-down);

.app-title{
    @include animate(fade-in-down, 2s);
}
    <!-- endtab -->
{% endtabbed_codeblock %}

Here you can see that this is a regular scss file but we have imported a module from the bower components folder and then we can use it in a simple way. The entry point for the generated css bundle is the site.scss file which contains only one line that imports the main.scss file. This file could be used as a global repository that loads all the other scss files required by the application.

#### Additional points

The current default template manages the predefined libraries in a way that could potentially implemented in a different way. For example the current implementation copies into the lib folder the packages that are hard copied inside the gulpfile and most specifically in copy command. This is not very flexible as it requires every time that a new dependency is added the modification of this file as well. A possible solution to this would be to require all this dependencies in the app.js file and let webpack to create the final bundle.

I have also include a gulp task that creates the minified versions of all the bundles that should always been used in production environment.

#### The next blog post

The next post which will come soon will expand this one and will describe how we can build a full isomorphic JavaScript application with asp.net vNext, [React.js][react], [Reactjs.net][reactnet] and [webpack][webpack].


[bootcamp]: http://global.azurebootcamp.net/
[skgazure]: http://www.skgazure.com/
[build]: http://www.buildwindows.com/
[channel9]: http://channel9.msdn.com/events/build/2015/
[devit]: http://devitconf.org/
[vnext]: http://www.asp.net/vnext
[npm]: https://www.npmjs.com/
[bower]: http://bower.io/
[gulp]: http://gulpjs.com/
[grunt]: http://gruntjs.com/
[node]: https://nodejs.org/
[aspdoc]: http://docs.asp.net/en/latest/index.html
[react]: http://facebook.github.io/react/
[webpack]: http://webpack.github.io/
[repo]: https://github.com/xabikos/aspvnextwebpack
[local]: http://localhost:5000/
[babelloader]: https://github.com/babel/babel-loader/
[expose]: https://github.com/webpack/expose-loader/
[animated]: https://github.com/jdsimcoe/animated.sass/
[reactnet]: http://reactjs.net/