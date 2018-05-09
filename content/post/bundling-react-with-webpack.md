+++
author = "Nick Lanng"
categories = ["javascript", "code"]
date = 2016-01-31T21:37:21Z
description = ""
draft = false
slug = "bundling-react-with-webpack"
tags = ["javascript", "code"]
title = "Bundling React with Webpack"

+++

A common complaint of the JavaScript ecosystem is the incredible rate of change.

> "Why bother learning any JavaScript frameworks when they'll just be defunct by next week?" - Tommy Developer

Tommy is kinda right. Every week there's a new popular Github project, and now some JavaScript frameworks have even started to deprecate themselves with a new release. I believe that you can mitigate this problem, though, by choosing a framework or library that embraces a larger architectural movement. This way, the specific implementation details may change but the concepts are transferable.

# What is React?

One of these architectural movements is bringing the web back to HTML tags. For many years, websites have looked like *'div soup'*, an incredibly complex tree of div elements with various classes on them to hook up styles and logic. Long gone are the days of HTML tags that mean anything,  a document model that actually represents the document.

**Web Components** is one solution to this problem, custom elements with behavior and logic all bundled together into a reusable component. Salesforce's **Lightning Components** are a similar idea. **React** also tries to solve this problem. 

React apps are written in JavaScript. React maintains a *virtual DOM* that keeps track of state changes between the rendered document and its own idea of what's happening. When React detects a change it updates the properties of the elements on the DOM. How this differs from other client-side frameworks is that React only has to scan its JavaScript based virtual DOM for changes and not the actual document DOM - this allows much faster scanning of changes. It also modifies the elements directly, it does not render the entire element again, which provides speed benefits over similar frameworks.

# What is Webpack?

A long time ago, if we wanted to separate our JavaScript code into different files, we'd have have to create some global namespace object to hold everything. How many times have you seen a file that starts with something like:

```javascript
namespace = namespace || {}
```

**Webpack** allows you to separate your JavaScript files in a more structured manner, and uses the same module system that **Browserify**, **Node.js** and **ECMAScript 6** use. Each file is in its own closure, so only things that you explicitly export are available to other files that require it.

Webpack also has the ability to package CSS and images into the same bundle and minify files for production.


# The Tutorial Part

### Install Webpack

The easiest way to get Webpack is to use **npm**, Node Package Manager. This command will install Webpack and make it available in your system's path.

```bash
npm install -g webpack
```

## Set Up the Project

We'll continue to use npm to get our other dependencies. First we must initialize our project:
```bash
npm init
```
The default for all of the prompts is fine for our purposes.

Our only run-time dependencies are React and its Dom manipulation package:
```bash
npm install --save react react-dom
```

We have a number of development dependencies:
```bash
npm install --save-dev webpack babel-core babel-loader babel-preset-es2015 babel-preset-react
```
We're going to be building our app with ECMAScript 6 features, so we need **Babel** to compile back down to ECMAScript 5, as not all the new features are widely supported yet. Here we install WebPack, Babel and additional dependencies to link the two together.

## Create a Simple React Component

Create a folder called **src** that will hold all of our development files. Inside **src**, create another directory called **components**. This folder will hold all of our React components. Create a new file called **HelloWorld.jsx** with the following code:

```jsx 
// HelloWorld.jsx
import React from 'react'

export default class HelloWorld extends React.Component {
  render () {
    return <h1>Hello World!</h1>
  }
}
```

This is a **jsx** file, which is React's version of JavaScript that allows inline HTML (not to be confused with an HTML file that allows inline JavaScript). This is the bit that normally makes developer's throw up, but just go with it - it really does make a lot of sense. Behind the scenes, this file is compiled down to plain old JavaScript and, if you really wanted to, you could write this file in plain old JavaScript but I promise you don't want to.

This is a simple component that simply renders out a header with the text 'Hello World' in it. Components can have more behavior that this, but for this demonstration this is adequate.

## Your Main JavaScript File

In the **src** folder, create a file called **main.js** with the following code:

```javascript 
// main.js
import React from 'react'
import ReactDom from 'react-dom'
import HelloWorld from './components/Helloworld.jsx'

const main = () => {
  const domHook = document.getElementById('app')
  ReactDom.render(<HelloWorld />, domHook)
}

main()
```

This is the entry point into the system. All of the rest of the JavaScript will hang off this file.

## Good Old Index.html

That's our JavaScript all done, now we need an actual webpage. As all of our HTML tags are defined as React Components, the HTML page is quite minimal. We need an element to hang React off and we need to include our bundled JavaScript.

Create a folder in the root called **dist**. This will be our deployable folder, create a file inside called **index.html** with the following content:

```html
<!-- index.html -->
<!doctype html>
<html>
  <body>
    <div id="app"></div>
    <script src="bundle.js"></script>
  </body>
</html>
```

But where the heck did **bundle.js** come from? That's the result of Webpack bundling all of our JavaScript together. It has our **main.js**, our **HelloWorld.jsx** and also the **entire React framework** all in one handy file.

## Bundling Our Files with Webpack

You don't *need* to create a configuration file but it is recommended. A lot of these options can be configured as command line arguments on the webpack executable but that would start to get unmanageable very quickly.

Create a file called **webpack.config.js** in the solution root.

```javascript
// webpack.config.js
const path = require('path')

const config = {
  entry: path.resolve(__dirname, 'src', 'main.js'), // the main entry point to the system
  output: {
    path: path.resolve(__dirname, 'dist'), // where to place the bundled file
    filename: 'bundle.js' // the name of the bundled file
  },
  module: {
    loaders: [{
      test: /\.jsx?$/, // a regex to choose which files to apply this loader to
      loader: 'babel', // we'll use the babel-loader
      query: {
        presets: ['react', 'es2015'] // choose which babel modes to use
      }
    }]
  }
}

module.exports = config
```

Now we can run Webpack and create a bundle.
In the root of your solution run:
```bash
webpack
```

This will start with **main.js** and traverse the imports to pull in all the required files into a new file called **bundle.js** and place it in **dist**.

If you want a minified version of the bundle simply run: 

```bash
webpack -p
```

## Final File Layout

<img src="/content/images/2016/01/react-webpack-files.png" alt="Final File Layout" style="width: 200px;"/>


# What This Means

Please use something like **Webpack**, or **Browserify**, or even **RequireJS** to pull together separate JavaScript files. Old techniques like the module pattern are implied, every file is a module and only exported things are available to consumers.

React is a great framework for building a UI and has a wide reaching set of companion technology such as **react-router** and **redux**. React can also sit within other frameworks like **Angular** so migrating can be done one component at a time from within an Angular app.

ECMAScript 6 features make JavaScript look almost unrecognizable and encourage a brilliant coding style - learn and use these features as soon as possible!