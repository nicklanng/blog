+++
author = "Nick Lanng"
categories = ["javascript", "code", "architecture", "coffeescript"]
date = 2015-11-13T16:12:42Z
description = ""
draft = false
slug = "basic-javascript-patterns"
tags = ["javascript", "code", "architecture", "coffeescript"]
title = "Basic Javascript Patterns"

+++

Javascript is quite a loose language, its possible to do things in a bunch of different ways. This tends to lead to developers coding themselves into a corner with custom patterns that seem clunky.

Below are a few of the patterns that are often seen in Javascript development, use these as a reference if you're unsure about how to write a Javascript module.

The **module.exports** property and the **require** function are found in either [Node.js](https://nodejs.org) or [Browserify](http://browserify.org/), I will follow up with a separate post on how to use Browserify to organise your client-side scripts.

## Class Pattern
Javascript isn't a class-based language, its a prototypal language. However, it can behave in similar ways - just try to remember that this isn't C#, it has its own behavior. What you may consider as WTFs are probably in line with the language specification.
{{< highlight js>}}
var Hello = (function() {

  // this is our object prototype and constructor
  function Hello(name) {
    this.name = name;
  }

  // this is a "static" function
  Hello.foo = function() {
    return "fuzzed up, " + this.name;
  };

  // this is an instance function
  Hello.prototype.bar = function() {
    return "beyond all repair, " + this.name;
  };

  // this is a private function
  var snafu = function() {
    return "situation normal, " + this.name;
  };

  // return the object prototype from the closure
  return Hello;

})();

module.exports = Hello;


// Usage in another module
var Hello = require('./Hello');
var hello = new Hello('Gunty McSquintlock');

console.log(Hello.foo()); // "fuzzed up, Hello" <- That's the name of the factory function!
console.log(hello.bar()); // "beyond all repair, Gunty McSquintlock"

console.log(hello.foo()); // error - not on the instance
console.log(hello.snafu()); // error - private
{{< /highlight >}}

## Module Pattern
In this pattern, private functions are declared inside the factory function and public functions are returned inside an object literal.
{{< highlight js>}}
// this is our factory
var Hello = function(name) {

  // this is a private function
  var snafu = function() {
    return 'situation normal, ' + name;
  };

  return {
    // this is an instance function
    bar: function() {
      return 'beyond all repair, ' + name;
    }
  };
};

// this is a "static" function
Hello.foo = function() {
  return 'fuzzed up, ' + this.name;
};

module.exports = Hello;

// Usage in another module
var Hello = require('./Hello');
var hello = Hello('Gunty McSquintlock');

console.log(Hello.foo()); // "fuzzed up, " <- Not inside a factory function, so there isn't a name!
console.log(hello.bar()); // "beyond all repair, Gunty McSquintlock"

console.log(hello.foo()); // error - not on the instance
console.log(hello.snafu()); // error - private
{{< /highlight >}}

## Revealing Module Pattern
This is similar to the Module Pattern, except every instance function starts life as a private function and then we decide which ones to reveal to the public return object.
{{< highlight js>}}
// this is our factory
var Hello = function(name) {

  // this is a private function
  var snafu = function() {
    return 'situation normal, ' + name;
  };

  // this *will* be an instance function, but looks like a private now
  var bar = function() {
    return 'beyond all repair, ' + name;
  };

  return {
    // reveal the instance function
    bar: bar
  };
};

// this is a "static" function
Hello.foo = function() {
  return 'fuzzed up, ' + this.name;
};

module.exports = Hello;

// Usage in another module
var Hello = require('./Hello');
var hello = Hello('Gunty McSquintlock');

console.log(Hello.foo()); // "fuzzed up, " <- Not inside a factory function, so there isn't a name!
console.log(hello.bar()); // "beyond all repair, Gunty McSquintlock"

console.log(hello.foo()); // error - not on the instance
console.log(hello.snafu()); // error - private
{{< /highlight >}}

--------

# The Coffeescript Version
I like Coffeescript, here are the Hello.js files in Coffeescript.

## Class Pattern
{{< highlight coffee>}}
# this is our class
class Hello
  # constructor
  constructor: (@name) ->

  # "static" function
  @foo: -> 'fuzzed up, ' + @name

  # instance function
  bar: -> 'beyond all repair, ' + @name

  # private function
  snafu = -> 'situation normal, ' + @name

module.exports = Hello
{{< /highlight >}}

## Module Pattern
{{< highlight coffee>}}
#  this is our factory
Hello = (name) ->
  # this is a private function
  snafu = -> 'situation normal, ' + name

  return {
    # this is an instance function
    bar: -> 'beyond all repair, ' + name
  }

# this is a "static" function
Hello.foo = -> 'fuzzed up, ' + @name

module.exports = Hello

{{< /highlight >}}

## Revealing Module Pattern
{{< highlight coffee>}}
# this is our factory
Hello = (name) ->
  # this is a private function
  snafu = -> 'situation normal, ' + name

  # this *will* be an instance function, but looks like a private now
  bar = -> 'beyond all repair, ' + name

  return {
    # reveal the instance function
    bar: bar
  }

# this is a "static" function
Hello.foo = -> 'fuzzed up, ' + @name

module.exports = Hello
{{< /highlight >}}
