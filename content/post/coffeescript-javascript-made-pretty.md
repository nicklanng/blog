+++
author = "Nick Lanng"
categories = ["code", "javascript", "coffeescript"]
date = 2015-06-14T14:57:49Z
description = ""
draft = false
slug = "coffeescript-javascript-made-pretty"
tags = ["code", "javascript", "coffeescript"]
title = "Coffeescript - Javascript Made Pretty"

+++

I've been writing a lot of Coffeescript lately. For a long time I avoided it, it looked all weird and scary but, once I finally bit the bullet, I realised its actually a rather pretty language.

Below I've highlighted some of my favourite parts of Coffeescript but it is in no way a complete guide. To learn more about Coffeescript, check out the excellent homepage at http://coffeescript.org/. You will find a detailed list of features, along with their Javascript alternatives, and also a neat sandbox for trying out the language and immediately seeing the output Javascript.

## Semi-colons
They are not used in Coffeescript, so that argument is avoided!

## String Substitution
As in Javascript, you can use **'** or **"** to declare a string.
Strings declared with **'** are literal, but string declared with **"** can have substituted values by using **#{ ... }**.

#### Coffeescript
{{< highlight coffee>}}
name = "Rufus"

'Hello #{name}!'
# output = Hello #{name}!

"Hello #{name}!"
# output = Hello Rufus!
{{< /highlight >}}

## Functions
Declaring a function in javascript is quite syntax heavy. 'Function' is a rather long word to see repeated in a code file and often whole lines are devoted to curly brackets. Coffeescript attempts to make this common declaration much more concise.

#### Javascript
{{< highlight js>}}
example = function() {
  return "someValue";
}

function example() {
  return "someValue";
}
{{< /highlight >}}

#### Coffeescript
{{< highlight coffee>}}
# in Coffeescript, the last line of a function is an implicit return
example = () ->
  "someValue"

# since there are no arguments to this function, the brackets are optional
example = ->
  "someValue"

# and why don't we just put it all on one line
example = -> "someValue"
{{< /highlight >}}

Calling functions is mostly the same, with the main difference being the optional use of brackets.

#### Coffeescript
{{< highlight coffee>}}
# as there are no arguments to this function, we must use
# empty brackets to show that we are not referencing the
# function but invoking it
example()

# when the function takes arguments, brackets are optional
anotherExample argument1, argument2
anotherExample(argument1, argument2)


# when chaining multiple function calls,
# it can make the code more readable to use occasional brackets
function1 function2 argument1, function3 argument2
function1 function2(argument1, function3(argument2))
{{< /highlight >}}

## Object Definitions
We don't use the curly brackets and commas. The colon syntax and indentation is enough to work out this this is an object and not code. Although, if you're declaring an object on one line, commas are needed.

#### Javascript
{{< highlight js>}}
example = {
  foo: "bar",
  sna: {
    description: "this looks like yaml"
  },
  someNumber: 42
};

someFunction({option: "aThing", choice: 3});
{{< /highlight >}}
#### Coffeescript
{{< highlight coffee>}}
example =
  foo: "bar"
  sna:
    description: "this looks like yaml"
  someNumber: 42

someFunction option: "aThing", choice: 3
{{< /highlight >}}

## 'Classes'
A lot of the features of Coffeescript classes are now available in ES6, but a couple of neat features aren't.

#### Javascript (Ecmascript 5)
{{< highlight js>}}
MyClass = (function() {
  function MyClass() {}

  var privateFunction = function() {
    return "I am not on the prototype of this object";
  };

  MyClass.prototype.myProperty = "whoopsydoo";

  MyClass.prototype.myFunction = function(argument) {
    return argument + " has been substituted into a string";
  };

  return MyClass;

})();
{{< /highlight >}}

#### Coffeescript
{{< highlight coffee>}}
class MyClass
  # note that this function is declared with an = not a :
  privateFunction = ->
    "I am not on the prototype of this object"

  myProperty: "whoopsydoo"

  myFunction: (argument) ->
    "#{argument} has been substituted into a string"
{{< /highlight >}}

**@** is a shortcut for **this** and can be used to save some more lines of code in constructors. These next two examples are functionally equivalent.

#### Javascript (Ecmascript 6)
{{< highlight js>}}
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
  toString() {
    return '(' + this.x + ', ' + this.y + ')';
  }
}
{{< /highlight >}}

#### Coffeescript
{{< highlight coffee>}}
class Point
  constructor: (@x, @y) ->
  toString: ->
    "(#{@x}, #{@y})"
{{< /highlight >}}

## Bound Functions
The **this** keyword can be a bitch. The meaning of **this** can change depending on what scope it is being used in - this certainly confused the hell out of me when I was learning Javascript and coming from the .Net world.

A common pattern in Javascript is to create a variable at the top of the class that saves the initial, class-scoped **this** value. Coffeescript supplies a shortcut to this pattern. When declaring a function you can use a fat arrow that passes the meaning of **this** into the function. An example below is from a class that takes a node socket and attaches an event handler to the data event. Note that the handler is created using the **=>** operator which passes in **this**.

#### Coffeescript
{{< highlight coffee>}}
cleanInput = (input) ->
  input.toString().replace(/(\r\n|\n|\r)/gm,"").toLowerCase()

class TelnetConnection extends EventEmitter
  constructor: (@socket) ->
    @socket.on 'data', (data) => # Fat arrow
      # this still refers to the TelnetConnection class
      @emit 'input', this, cleanInput(data)
{{< /highlight >}}

# You should learn Javascript first
I'd be wary of any developer starting out with Coffeescript without first having a good understanding of Javascript. Coffeescript is transpiled into Javascript before being run so if you get a crazy error message you're gonna want to know what it means. Particularly when learning, you will write malformed Coffeescript that transpiles into perfect Javascript, but not in the way you intended.
There are tools out there that let you preview the Javascript generated by your Coffeescript, but they're only helpful if you know what the Javascript is supposed to look like!

## Coffeescript? Typescript? Dart? Javascript?
Javascript, yes. If you are a web developer, learn Javascript.

As for the other three, it doesn't really matter. I think certain cultures value the various Javascript-like languages differently so you're free to choose the one you like or the one that the people you want to fit in with like.

I like Coffeescript because it is pretty and concise. Others like Typescript because of the type safety and really great Microsoft support. I don't really know much about Dart.
