+++
author = "Nick Lanng"
categories = ["javascript", "code"]
date = 2016-01-09T17:58:50Z
description = ""
draft = false
slug = "array-reduce-the-daddy-of-array-manipulation"
tags = ["javascript", "code"]
title = "Array.prototype.reduce - The Daddy of JavaScript Array Manipulation"

+++

Shout out to **mpjme**'s YouTube channel that inspired this post - see the video here: https://www.youtube.com/watch?v=Wl98eZpkp-c

## What is Array.prototype.reduce?

**Array.prototype.reduce** is a function on the **Array** prototype in JavaScript. Its purpose is to do some sort of operation on each element in an array and then return a result of some type.

```javascript
arr.reduce(callback[, initialValue])
```

The **initialValue** is optional. It is often things like **0**, **[]** or **{}** but it can be whatever you like.

The callback is a function with the following arguments:

1. **previousValue** - the value returns from the last iteration
2. **currentValue** - the item in the array we're currently operating on
3. **index** - the index of the current item in the array
4. **array** - the array itself we are iterating

Check out more detailed specs here: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/Reduce

## Let's have some fun

Lately, I've been using the following **Array** functions a hell of a lot:

* **Array.prototype.map** // transform an array into another array of the same length
* **Array.prototype.filter** // get a subset of the array with elements that match the filter function
* **Array.prototype.reject** // the opposite of Array.filter

If you're a fan of **F#** or **Linq** in .Net or a multitude of other functional bits and bobs these should be familiar.

Let's try to write our own version of these functions using **Array.prototype.reduce** as a starting block:

```javascript
const map = (array, mapping) =>
  array.reduce((result, item) => {
    result.push(mapping(item)) // run the map function on the current item and add it to the new array
    return result
  }, []) // an empty array is our initial value

const filter = (array, criteria) =>
  array.reduce((result, item) => {
    if (criteria(item)) result.push(item) // if the criteria function result is true, add this item to the new array
    return result
  }, []) // an empty array is our initial value

const reject = (array, criteria) =>
  array.reduce((result, item) => {
    if (!criteria(item)) result.push(item) // if the criteria function result is false then do not add that item to the new array
    return result
  }, []) // an empty array is our initial value
```

### The usage example
```javascript
const orders = [
  { amount: 250 },
  { amount: 400 },
  { amount: 100 },
  { amount: 325 }
]

const orderAmounts = map(orders, item => item.amount)
console.log(orderAmounts) // [ 250, 400, 100, 325 ]

const ordersOver300 = filter(orders, item => item.amount >= 300)
console.log(ordersOver300) // [ { amount: 400 }, { amount: 325 } ]

const orderUnder300 = reject(orders, item => item.amount >= 300)
console.log(orderUnder300) // [ { amount: 250 }, { amount: 100 } ]
```