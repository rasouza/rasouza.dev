---
layout: post
title: "Optimize your JS code using Google's V8 internals"
date: 2018-09-14
category:
  - Post
image: js-v8.png
author: Rodrigo Souza
tags:
  - EN
  - Javascript
---
At BrazilJS 2018 this year in Porto Alegre, [Mathias Bynens](https://github.com/mathiasbynens) (a V8 developer at Google) shared some insights for code optimization if you are using (and you probably are) the V8 engine for Javascript. Here's what I learned.

For those who haven't heard of it, **Google's V8** is an open source JavaScript engine written in C++ developed by **The Chromium Project** for Chromium and Google Chrome web browser. It means that when you use Chrome and it detects JavaScript code, it passes that code to V8 to run the compilation and then your computer executes the result. It's also the cornerstone of the popular backend framework called [Node.js](https://nodejs.org/en/).

In this post, we're going to see how V8 optimizes your code using a primitive called element kinds.

When you run JavaScript and assign variables in it, the V8 engine keeps track of the value types you're saving in memory besides the value itself. These *value types *are the element kinds we're talking about. Take this array as example:

```js
const myArr = [1, 2, 3]
```

> At the language-level, it's all `Numbers` to JavaScript for it doesn't make distinction between integers, floats or doubles. But internally it does much more than that: V8 also identifies that it contains only integers and assign `PACKED_SMI_ELEMENTS` to `myArr`.

Forget `PACKED_` portion for now...

Common Element Kinds
====================

`SMI_ELEMENTS` means it contains only SMall Integers and it can make optimizations when you're accessing it later. However, as soon as you add `3.14` to that array, it becomes `PACKED_DOUBLES_ELEMENTS` immediately and it can never go back to `PACKED_SMI_ELEMENTS` no matter what you do. If you go further and add `'x'` it is demoted to `PACKED_ELEMENTS` and becomes even more generic!

```js
const array = [1, 2, 3];
// elements kind: PACKED_SMI_ELEMENTS
array.push(3.14);
// elements kind: PACKED_DOUBLE_ELEMENTS
array.push('x');
// elements kind: PACKED_ELEMENTS
```

That's the first thing you have to keep in mind:

Rule 1: Always try to be as high as possible in the pyramid of element kinds
----------------------------------------------------------------------------

![](https://miro.medium.com/max/1082/1*Cp1eYJMl_eTSsZMYldWhHg.png)

<em>Element kinds pyramid used to classify array of objects in JavaScript. The higher the objects are in pyramid the more optimized it is when doing operations on it afterwards. (got from [Mathias Bynens's slides](https://mathiasbynens.be/))</em>

Before digging into how V8 makes these optimizations, let's unbox that `PACKED_` portion we left aside.

Holey vs Packed
===============

So far, our array values are `[1, 2, 3, 3.14, 'x']` and is considered by V8 engine as `PACKED_ELEMENTS` at this point. Now, consider this line of code:

```js
myArr[9] = 1
```

![](https://miro.medium.com/max/1400/1*qzgHNeTWglPZt1B5Cuifrw.png)

<em>Our array has holes from index 5 to 8</em>

We've demoted `myArr` even more. It's now considered a `HOLEY_ELEMENTS` since it contains `undefined` values and it can never become`PACKED` again. So we end up with this elements kinds lattice.


![](https://miro.medium.com/max/1280/1*UVqMOw7tO7_IqQNyiqiJQg.png)

<em>Elements Kinds Lattice (Source: <https://v8project.blogspot.com/2017/09/elements-kinds-in-v8.html>)</em>

It's only possible to transition *downwards*. If you want to have `PACKED_SMI_ELEMENTS` again you must create a brand new array containing only integers with no holes in it. Hence...

Rule 2: Always try to make your elements packed from the very beginning
-----------------------------------------------------------------------

Take a look at this piece of code:

```js
const array = new Array(3); // HOLEY_SMI_ELEMENTSarray[0] = 'a'; // HOLEY_ELEMENTS\
array[1] = 'b'; // HOLEY_ELEMENTS\
array[2] = 'c'; // HOLEY_ELEMENTS (still!)
```

![](https://miro.medium.com/max/600/1*2Wbyqt0vezFqu2egvfRcOA.png)

<em>Array values at the end of the code</em>

Even though `array` is fulfilled, it's not possible to transition back to `PACKED`. It's too late and `array` is still a `HOLEY_ELEMENTS` kind.

Optimizations done in V8
========================

Let's go for the fun part! And for this, I'm going to propose something different, let's invert the roles now: imagine you are the developer behind V8 engine and you must make V8 respond properly to JS code (come on you can do it!). Let's recap:

> `myArr` is `[1, 2, 3, 3.14, 'x', undefined, undefined, undefined, undefined, 1]`. Indexes 5--8 are `undefined` because of the hole we created instead of having something assigned (we could assign `undefined` for example). Therefore there's nothing in its memory addresses but we know JavaScript must return `undefined` in this case.
>
> I want to access `myArr[8]` which is a `HOLEY_ELEMENTS`. What you must do to ensure the return is `undefined` given you don't know much about `myArr` structure and you don't have this value properly saved in memory address?

Don't be so harsh on yourself and take some time to think about it... Ok, time's up!

If you guessed correctly, Google's V8 engine makes the following checks to make sure the value in `myArr[8]` is `undefined` considering it's a `HOLEY_ELEMENTS` kind:

1.  `8 >= 0 && 8 < myArr.length` (boundary checks) returns true. So it's allowed to continue...
2.  `hasOwnProperty(myArr, 8)` returns false. Nope, it's not here, there's a couple of checks left to do...
3.  `hasOwnProperty(Array.prototype, 8)` returns false. Not here also, let's try somewhere else...
4.  `hasOwnProperty(Object.prototype, 8)` returns false. Cannot find it anywhere. OK, let's return `undefined` then.
5.  Return `undefined`.

Seems like a bunch of duplicate checks, right? And, yes, it's kind of hard to guess straightaway. Now, let's take a look how it works for a different element kind. Consider the same example but for `PACKED_ELEMENTS`:

```js
const packedArr = [42, undefined, 42];\
packedArr[1];
```

V8 engine makes fewer checks:

1.  `8 >= 0 && 8 < packedIntArr.length` returns true. Since it's `PACKED`, the value surely exists, you can return whatever you find in memory.
2.  Return `undefined`.

> Even though the return value is the same, V8 has done far less work behind the scenes. Therefore, you may want to keep your data as packed as possible as well.

This performance also impacts on several other common operations:

```
Array.prototype.forEach
Array.prototype.map
Array.prototype.filter
Array.prototype.some
Array.prototype.every
Array.prototype.reduce
Array.prototype.reduceRight
Array#{find, findIndex}
```

> Since Chrome version 70 all the operations listed above are already optimized inside their element kind. This means that if you want better performance, keep your data structure similar to its kind as possible.

Last but not least...

Rule 3: When you're iterating through your collection, avoid out-of-bound reads
-------------------------------------------------------------------------------

Use whatever you prefer: `array.forEach` or `item of array` (both has the same performance nowadays) but avoid reaching out of bounds indexes like the code below:

```js
for (let i = 0, item; (item = array[i]) != null; i++) { 
  doSomething(); 
}
```

This pretty much concludes this type of optimization in the V8 engine. Of course, there are not only 6 elements kinds. There are [21 different elements kinds](https://cs.chromium.org/chromium/src/v8/src/elements-kind.h?l=14&rcl=ec37390b2ba2b4051f46f153a8cc179ed4656f5d) and each kind is optimized its own way, but I'm sure you'll start to pay more attention on how you define your data structure after reading this post through.

TL;DR
=====

In order to let V8 engine make better optimizations on your JavaScript code, you may want to follow some guidelines when writing it:

-   Avoid element kind transitions
-   Avoid holes
-   Avoid out-of-bounds reads
-   Prefer arrays over array-like objects

All in all, write modern, idiomatic JavaScript and let the JavaScript engine worry about making it fast. Thanks Mathias to for the awesome talk at BrazilJS and for sharing your knowledge!