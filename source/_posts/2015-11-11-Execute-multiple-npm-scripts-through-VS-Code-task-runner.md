---
title: Execute multiple npm scripts through VS Code task runner
date: 2015-11-11 20:40:00
tags: [nodejs]
categories: [npm scripts, task runner]
alias: npm scripts/task runner/2015/11/11/execute-multiple-npm-scripts-through-vs-code-task-runner.html
keywords:
- node.js
- npm scripts
- vscode
---

#### VS Code

[Vs code][code] is one of the new players in the area of editors and has done some really good progres so far on catching up with the competition. It is a lightweight editor which currently supports [mulitple langueges][languages] and on top of that it's [multiplatform][multiplatform]. It includes already some great features and more are coming in the near future. Personally I am using it more and more both for front and back end development.
<!-- more -->
#### Webpack - Gulp - npm scripts

During the last months I am heavily involved in various Reactjs project so there is a need to execute scripts in order to transpile React components and ES6 code. In all projects we are doing heavy usage of [Webpack][webpack] in order to manage the transpilation and the dependencies management. Initially we were doing all the configuration through [Gulp][gulp] and it's corresponding tasks. As Webpack evolved and I was more involved in nodejs ecosystem I found the npm scripts and how by using this feature we could avoid the dependency to Gulp or any other task runner.

#### VS Code tasks for the rescue

Code has integrated the ability to execute various tasks as you can see in the official [documentation][tasks]. As it mentioned clearly in the documentation there is already integrated support for Gulp, Grunt and some other tools. What is missing is that with a few lines of configuration we could execute all the scripts that exist in the corresponding section of our package.json file.

#### Create the tasks.json file

Available tasks are declared in an external file called "tasks.json" and is placed inside the folder .vscode in the root of the project. You can either create it manual or let the editor create the folder and add a very detailed example file. In order to achieve the last you need to hit Ctrl + Shift + P or Cmd + Shift + P type "Tasks" and select Configure Task Runner as in the following image

{% image fancybox nocaption fig-100 clear group:tasks configureTasks.png "configure tasks" %}

#### Add the configuration for npm scripts

In order to be able to execute the npm scripts we need to tweak the first task that is related to Typescript. As a first step lets see what are the available scripts inside the package.json file.

{% tabbed_codeblock npm scripts %}
    <!-- tab json -->
"scripts": {
  "build-dev": "webpack --bail",
  "start-dev": "webpack-dev-server"
},
    <!-- endtab -->
{% endtabbed_codeblock %}

As I mentioned before we are doing a lot of react development so these two scripts are responsible for building the development version of the code the first and for starting the webpack's development server and watching the files for changes the second. Now lets see how the tasks.json file looks like

{% tabbed_codeblock Add npm as comand %}
    <!-- tab json -->
{
  "version": "0.1.0",
  "command": "npm",
  "isShellCommand": true,
  "showOutput": "always",
  "args": ["run"],
  "isWatching": false,
  "tasks": [
    { "taskName": "build-dev" },
    { "taskName": "start-dev"	}
  ]
}
    <!-- endtab -->
{% endtabbed_codeblock %}

As you can see we have to declare "npm" as the required command and configure it as a shell command. The crucial part is the args option as all the scripts are executed by "npm run" as prefix so we need to provide this as an argument. The show output parameter is entirely up to you how to configure it. Then we need to declare the tasks for the command inside the corresponding array. Each task can have its own configuration and arguments but I would suggest not to use this option and instead putt all required arguments to the scripts section as some other members in the team could execute these from another tool. So in our case is enough to declare just the tasks name which of course should match with the one in scripts section.

#### Task execution

Now that everything is in place when hitting Ctrl + Shift + P or Cmd + Shift + P and select "Run Task" we should see a list with the available tasks that represent the scripts. We can choose either of the options and depending on the show output configuration we will be able to see the results in the output window.

{% image fancybox nocaption fig-100 clear group:tasks availabletasks.png "availalbe tasks" %}

#### Bonus - tasks keyboard shortcuts

Be default VS Code has assigned keyboard shortcuts for execute the build and test task. Luckily there are available the extension points for showing the panel with available tasks and also a command to stop the running task. First you need to find the preferences and then select Keyboard Shortcuts and then add the following entries. Feel free to change the key combinations according to your preferences. It was hard though to find some combination so I chose r for run a task and e for exit a task. After saving we can have an easier access to both select the command and also to terminate it.

{% tabbed_codeblock Keyboard shortcuts %}
    <!-- tab json -->
{ "key": "shift+cmd+r",  "command": "workbench.action.tasks.runTask" },
{ "key": "shift+cmd+e",  "command": "workbench.action.tasks.terminate"}
    <!-- endtab -->
{% endtabbed_codeblock %}

Happy coding!

[code]: https://code.visualstudio.com
[languages]: https://code.visualstudio.com/docs/languages/overview
[multiplatform]: https://code.visualstudio.com/docs/editor/setup
[webpack]: http://webpack.github.io
[gulp]: http://gulpjs.com
[tasks]: https://code.visualstudio.com/docs/editor/tasks