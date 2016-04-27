---
title: Webpack aliases and relative paths
date: 2015-10-03 12:10:00
tags: [webpack]
categories: [javascript module bundler, javascript dependencies management]
alias: javascript module bundler/javascript dependencies management/2015/10/03/webpack-aliases-and-relative-paths.html
keywords:
- webpack
---


#### Why Webpack

[Webpack][webpack] is one of the solutions that are available out there and solves JavaScript reference modules problem. It's also my favourite tool to use it in the Reactjs project I am involved lately. Webpack provides wider functionality except just referencing external JavaScript code but this is not the scope of this blog spot. In this post I will try to concentrate in a feature of the tool that most examples don't use it and it's very important according to my opinion. This is the [resolve][resolve] section of Webpack configuration and how it can help us in day to day development.
<!-- more -->
#### An example without using aliases

As mentioned earlier I am involved lately in multiple projects related to Reactjs so the example uses this library. What is described on the other hand could easily applied in frameworks like Angular or just pure JavaScript code which is bundled through Webpack.

Let's say that we have a really simple components hierarchy that one uses the other. A view of this example could be this one:
{% tabbed_codeblock Folder structure %}
    <!-- tab -->
<App>
  <Utility>
    <Home>
      <Utility>
      <Service>
    <!-- endtab -->
{% endtabbed_codeblock %}
That means we have a Utility component (you could consider it a separate file) that is used by both App and Home component and on top of this Home component uses also an external service. This is reflected in the file system as:

{% image fancybox nocaption fig-33 center clear group:webpack FolderStructure.png "folder structure" %}

Even in new ES6 module system when we need a reference in a piece of code in our project that is not a node module, we need to reference it by relative path to the file we need it. So to make it more concrete the import code for App component looks like this:

{% tabbed_codeblock Import Utility module %}
    <!-- tab js -->
import Home from './home.jsx';
import Utility from './common/utility.jsx';
    <!-- endtab -->
{% endtabbed_codeblock %}

And to make the issue with relative paths more obvious lets see the import statements of Home component:

{% tabbed_codeblock Import modules relative paths %}
    <!-- tab js -->
import Utility from './common/utility.jsx';
import TextService from '../services/textService.js';
    <!-- endtab -->
{% endtabbed_codeblock %}

From the example above we should realise that each time we need to use a piece of code in our application, we need to now in which exact folder we are and then start calculating the relative path to this file. This is something that really annoys me and is not only that. Just think the case that the project evolves and the utility file is referenced in multiple places. And then we need to either rename the file or more probably reorganise the folder structure and choose a different one. We have to find all the files that reference this file and make the appropriate changes.

#### Webpack Resolve Alias solution

As mention before [Webpack][webpack] has a section in it's configuration ([resolve section][resolve]) that could solve this issue and it's very powerful. We can declare aliases in many ways so we could not rely on relative paths but just reference our code as regular node modules.

First let's see what we need to add to the webpack.config file:

{% tabbed_codeblock Webpack alias config %}
    <!-- tab js -->
resolve: {
  root: path.resolve(__dirname),
  alias: {
    app: 'components/app',
    home: 'components/home',
    utility: 'components/common/utility',
    textService: 'services/textService'
  },
  extensions: ['', '.js', '.jsx']
},
    <!-- endtab -->
{% endtabbed_codeblock %}

It's important to configure the root with an absolute path so the path in the alias section is becoming relative to where the webpack.config file resides. Of course in a real project where we have more files we should probably extract this object to a separate file and import it in webpack.config file. After that is our responsibility to have unique names in aliases something that probably was easier before, due to folder names.

But now is interesting to see how the import statements of components look like. Let's begin with App component as before:

{% tabbed_codeblock Import module through alias %}
    <!-- tab js -->
import Home from 'home';
import Utility from 'utility';
    <!-- endtab -->
{% endtabbed_codeblock %}

And let's also see the Home component

{% tabbed_codeblock Import module through alias %}
    <!-- tab js -->
import Utility from 'utility';
import TextService from 'textService';
    <!-- endtab -->
{% endtabbed_codeblock %}

By now should be obvious that it's easier to reference any file in our project. On top of that, where every file resides exists only in one place so if there is a need to either rename a file or restructure the folders the only place we have to touch is the aliases. In the configuration I also added the extensions option as it's a way not to include the extension of the file in the path.

You can find the code for this example in the corresponding [Github repository][githubproject].

[webpack]: http://webpack.github.io
[resolve]: https://webpack.github.io/docs/configuration.html#resolve
[githubproject]: https://github.com/xabikos/webpack-alias