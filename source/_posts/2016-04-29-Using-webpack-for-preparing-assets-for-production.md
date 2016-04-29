---
title: Using webpack for preparing assets for production
date: 2016-04-29 20:55:06
tags: [webpack, production asset management]
categories: [javascript module bundler, javascript dependencies management, css module bundler, css dependencies management]
keywords:
- webpack
---

## Static asset management in modern web development

Since the beginning of web development static asset management and especially for production environment is a headache for the developers as there are so many options and various choices we can make. When we discuss about static assets we mean mainly JavaScript, CSS and image files. Among various tools and methodologies, we choose to use [Webpack](https://webpack.github.io/) as it is a very powerful and flexible piece of software that among other usages, it is capable of preparing the production bundles. In the rest of the tutorial we won't cover any of Webpack's feature that are used during development. You can find [here](https://www.codementor.io/reactjs/tutorial/beginner-guide-setup-reactjs-environment-npm-babel-6-webpack) an excellent post about setting an environment for React with npm, Babel 6 and webpack.
<!-- more -->

### A different configuration file

Webpack is just another node module that can be installed either locally or globally in your system. It can be configured both with [command line arguments](http://webpack.github.io/docs/cli.html) and an [external configuration file](http://webpack.github.io/docs/configuration.html). In our case we will go for the external configuration file. It's a good idea to have two different configuration files for webpack, one that is used during development and one for production environment. A common practice is to call this file `webpack.production.config.js` but feel free to name it according to your preference. Now to be able to execute webpack with this configuration file we need to pass the corresponding command line argument 
```
webpack --config  webpack.production.config.js
```
 if we have installe it globally or
```
node_modules\.bin\webpack --config  webpack.production.config.js
```
 in case the tool is only installed locally to the project.

### The example application

I have created a small application which is hosted in [Github](https://github.com/xabikos/webpack-for-production) in order to have a concrete example and apply the configuration on it. The application uses [React](https://facebook.github.io/react/) as a front end technology but keep in mind that webpack can be used with all popular front end libraries and frameworks. The example application is really simple and all it does is allow us to query Github's repositories api for a term and filter based on the language.
![GitHub repo explorer](https://www.filepicker.io/api/file/eNqtHxPOSlKqHAKe2WCN "Github explorer")

During the rest of the tutorial we will explain how webpack can take care first of the JavaScript files and then for styles and images.

The repository contains several tags that represent the several steps of the tutorial that follow. 

## Initial configuration
When checking out the tag with name "initial" we have the starting point of the configuration that probably looks very close to the one we use during development. In order to have a concrete view of the outcome we can execute 
```
npm run dist
```
and then open the dist folder that is created automatically. This folder contains only two files one that is called bundle.js and contains the entire code for the application and an index.html file that references the bundle. 

We need to mention at that point that the generated file contains both the JavaScript and styles 
we added but also [bootstrap's](http://v4-alpha.getbootstrap.com/) code and styles as it's used internally in the application. Let's see now how we can improve the generation of that file.

## Extract styles
The first improvement is to extract the styles in a separate css file. To achieve this we are going to use the [ExtractTextPlugin](https://github.com/webpack/extract-text-webpack-plugin). After making the required changes to webpack's configuration file and execute 
```
npm run dist
```
once again we should have in our dist folder three files that look like in the following image
![separate js and styles](https://www.filepicker.io/api/file/5MdcxES0Tv64kSFSitl3 "files list")
At this point the style.css file contains both our styles but also bootstrap's styles too and the same applies for the javascript file.

## Minification 
The first action that we can apply in order to reduce the size of the files is to minify them. To make this happen we are using the [UglifyJsPlugin](https://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin). After that change we notice a tremendous reduction on the size especially for the bundle.js
![minified files](https://www.filepicker.io/api/file/ptXclda5R4mGjBvORhAi "file list after minification")

## Split vendor code
A nice approach when using npm for development is to always separate [dependencies](https://docs.npmjs.com/files/package.json#dependencies) and [devdependencies](https://docs.npmjs.com/files/package.json#devdependencies). Among the benefits that are described in the official documentation this distinction can help us split our JavaScript code in two main files, one file for library or vendor and the other that contains our application's code. This is a best practice as in every new deployment of the app the user needs to download the updated files and it's more common that there are only changes in ours code rather than the libraries. So the user needs to download every time a smaller file that contains our code and the larger vendor file when we update one of our dependencies. To achieve that we need to configure the [CommonsChunkPlugin](https://webpack.github.io/docs/list-of-plugins.html#commonschunkplugin) and the result is reflected in the following image
![file list after vendor](https://www.filepicker.io/api/file/MW9oTSs0TsuGZAKgID5L "file list after vendor")

## Cache busting
Webpack is able also to handle the cache busting problem between new deployments. After applying the required configuration our list of files looks like this
![add cache busting](https://www.filepicker.io/api/file/hnCA5opLRaDr7i5QEr0Z "file list after cache busting addition")

## Additional optimizations
We can further reduce the size of the file by applying some additional plugins. First we can use the [DefinePlugin](https://webpack.github.io/docs/list-of-plugins.html#defineplugin) in order to correctly set the **NODE_ENV** environment variable to "production" that is highly recommended to do especially for React applications. On top of that we should use the [DedupePlugin](https://webpack.github.io/docs/list-of-plugins.html#dedupeplugin) and [OccurrenceOrderPlugin](https://webpack.github.io/docs/list-of-plugins.html#occurrenceorderplugin) that help to optimize and further reduce the final size of the files as we can see in the image that follows
![additional plugins](https://www.filepicker.io/api/file/KKw0BuASSICd8nL6CJio "file list after applying additional optimizations")

## Image handling
In our example application we reference two images one small that is shown in the top right corner and one larger in the footer of the application. Image storing into folders and reference it in our components is a hard story and it's very nice that webpack is able to assist in that section too. This can be achieved by requiring the images as other files and then let webpack to copy them to the appropriate destination folder. For example inside Footer component the code from 
```
<img className="center-block" src="/images/footer.jpg" />
```
should be changed to 
```
import footerImage from '../images/footer.jpg';

<img className="center-block" src={footerImage} />
```
After the changes we need to configure the [url-loader](https://github.com/webpack/url-loader) to handle the images.
```
{
  test: /\.(png|jpg|jpeg|gif|svg|woff|woff2)$/,
  loader: "url?limit=10000"
}
```
As we can see here this loader is able to handle any kind of static resources. All the images that are below the specified limit are embedded inside the css file as base64 strings and the rest are delivered to file loader and are just copied in the destination folder. After these changes our dist folder looks like this ![files with images](https://www.filepicker.io/api/file/jFxsBKo0QVWgd1qsRlxA "File list with images")
In our case the larger jpg file is just copied to the destination folder and the smaller exists inside the minified version of the css something that reduces the number of the requests to the server.

## Linting the code
Of course linting the JavaScript code must be part of the production build and webpack can take care of that too by configuring the necessary loader. In our example we are using [eslint](http://eslint.org/) and the corresponding [eslint-loader](https://github.com/MoOx/eslint-loader). We need to highlight here that we configure this as a preloader as we prefer to break the build as soon as possible of a linting error is present.
```
preLoaders: [
  {
    test: /\.jsx?$/,
    loader:'eslint',
    exclude: /node_modules/
  }
],
```

## Summury
In this article we tried to demonstrate how webpack can assist us with static resources management in modern web development. Webpack is a very powerful tool and there a re further optimizations we can use to improve the performance of our application.
