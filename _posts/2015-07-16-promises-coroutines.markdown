---
layout: post
title:  "Combining Promises and Generators to write Elegant Asynchronous code in Javascript"
date:   2015-08-01 14:04:21
categories: jekyll update
---

Introduction
==============

The purpose of this blog post is to teach you how promises and generators can be combined to write asynchronous code in Javascript that *looks* synchronous. Writing code this way makes it very easy to read and reason about what the code is doing. I'm not going to cover the topic of promises because there are plenty of resources online that provide much better explanations than I could. I will, however, provide a brief overview of generators in Javascript, but I recommend that you pursue additional reading elsewhere.

Generators
==========

Generators were added to Javascript as a result of ES6. A generator in JavaScript is a special type of function that can halt itself at any time using the keyword **yield**, and then later resume execution. Whats so interesting about generators is that they allows a developer to write a function that not only can **"return"** multiple times, but can also have values **"injected"** into it mid-execution.

If that sounded confusing, don't worry, generators quickly begin to make sense once you start playing around with them. That said, open up your favorite ES6-compliant* Javascript REPL and paste in the following code:

{% highlight javascript linenos %}
/*the asterisk indicates that the function is a 
generator instead of a traditional function */
var generatorFactory = function* () {
  var result1 = yield 1;
  console.log(result1);
  var result2 = yield 2;
  console.log(result2);
  var result3 = yield 3;
  console.log(result3);
}

var testGenerator = new generatorFactory();
{% endhighlight %}

Now try calling testGenerator.next() a few times. You should get something that looks like this:

{% highlight javascript linenos %}
console.log(testGenerator.next()); 
// Object {value: 1, done: false}
console.log(testGenerator.next()); 
// undefined
// Object {value: 2, done: false}
console.log(testGenerator.next());
// undefined
// Object {value: 3, done: false}
console.log(testGenerator.next());
// undefined
// Object {value: undefined, done: true}
{% endhighlight %}

Everytime you call testGenerator.next(), the function executes until it encounters a yield statement. The `yield` statement functions like `return` in that the invoking function will evaluate to whatever value directly follows the keyword `yield` (except it will be wrapped in a container object, but we'll get to that in a moment). The first time you call `testGenerator.next()`, the `generatorFactory` function will execute until it encounters the first `yield` statement:

{% highlight javascript %}
var result1 = yield 1;
{% endhighlight %}

At this point the generator will **yield** (pausing execution) and the statement `testGenerator.next()` will evaluate to an object that contains two properties:
  
1. value: the yielded value
2. done: a boolean value that represents whether or not the generator has any more code to execute (false in this case) 

The generator will remain paused until the next time `testGenerator.next()` is called. This process repeats itself everytime we call `testGenerator.next()` until there are no more `yield` statements, at which point the done property will be true.

But why did all the console logs evaluate to undefined? When `.next()` is called, the yield statement is replaced with whatever parameter is passed to `.next()`. This is how values can be **"injected"** into generators mid-execution. To see this in action, create a new `testGenerator` and then execute the following code:

{% highlight javascript linenos %}
console.log(testGenerator.next()); 
// Object {value: 1, done: false}
console.log(testGenerator.next(4)); 
// 4
// Object {value: 2, done: false}
console.log(testGenerator.next(5));
// 5
// Object {value: 3, done: false}
console.log(testGenerator.next(6));
// 6
// Object {value: undefined, done: true}
{% endhighlight %}

The values that are passed into `.next()` as parameters are stored in the `result` variables, and subsequently console logged.

Combining Generators and Promises
=================================

Ok, now that we have a basic understand of promises and generators, how can we combine these to write elegant and easy-to-understand asychronous code? Well first, lets take a look at the problem we're trying to solve:

{% highlight javascript linenos %}
var ajaxFunction = function(val) {
  promisifiedAjaxCall(val).then(function(result1) {
    anotherPromisifiedAjaxCall().then(function(result2) {
      if (result1 === someValue && result2 === someOtherValue) {
        doSomeThing();
      }
      else {
        doSomeOtherThing();
      }
    });
  });
});
ajaxRoutine(val);
{% endhighlight %}

Even with only two asynchronous calls, this function is starting to get a little messy, and we're not even handling errors yet! Lets try rewriting this function using a combination of promises and a generators:

{% highlight javascript linenos %}
var ajaxGenerator = function*(val) {
  var result1 = yield promisifiedAjaxCall(val);
  var result2 = yield anotherPromisifiedAjaxCall();
  if (result1 === someValue && result2 === someOtherValue) {
    doSomeThing();
  }
  else {
    doSomeOtherThing();
  }
}
var testGenerator = new ajaxGenerator(val);
var result1Promise = testGenerator.next().value.then(function(result) {
  testGenerator.next(result);
});
var result2Promise = testGenerator.next().value.then(function(result) {
  testGenerator.next(result);
});
{% endhighlight %}

So whats going on here? Lets walk through how this code will execute. The first time `testGenerator.next()` is called on line 12, `ajaxGenerator` will run until it hits the first `yield` statement on line 2.

{% highlight javascript %}
var result1 = yield promisifiedAjaxCall(val);
{% endhighlight %}

`result1Promise` variable will then be assigned a promise. Once the ajax call is complete and the promise resolves, the `.then()` function will execute, and the response from the ajax call will be passed back into the `ajaxGenerator` function (as a result of invoking `.next()`) and assigned to `result1`.

{% highlight javascript %}
var result1Promise = testGenerator.next().value.then(function(result) {
  testGenerator.next(result);
});
{% endhighlight %}

The code will continue to execute until it hits the `yield` statement on line 3 at which point the process will repeat, eventually assigning the result of the second ajax call to `result2`. The code can now be thought of as looking like this:

{% highlight javascript linenos %}
var ajaxRoutine = function*(val) {
  var result1 = resultFromFirstAjaxCall;
  var result2 = resultFromSecondAjaxCall;
  if (result1 === someValue && result2 === someOtherValue) {
    doSomeThing();
  }
  else {
    doSomeOtherThing();
  }
}
{% endhighlight %}

Now that the ajax calls have been resolved, its very easy to reason about the remaining code in linear fashion as you would with any synchronous functon. This is nice, however, actually invoking the function and feeding the resolved promises back into the generator is a messy process. What we want is something that not only can yield promises, but can automatically wait for them to resolve, feed their values back into itself, and then resume execution. Luckily, the Bluebird promise library has such a function. The code snippet above can be rewritten like this:

{% highlight javascript linenos %}
var Promise = require('bluebird');

var ajaxRoutine = Promise.coroutine(function*(val) {
  var result1 = yield promisifiedAjaxCall();
  var result2 = yield anotherPromisifiedAjaxCall();
  if (result1 === someValue && result2 === someOtherValue) {
    doSomeThing();
  }
  else {
    doSomeOtherThing();
  }
});

ajaxRoutine(val).catch(function(err) {
  if (err) throw err;
});
{% endhighlight %}

This time, the same process will occur and the code will execute asynchronously, waiting for each ajax request to complete before continuing, but the code can be read and reasoned about in a synchronous manner. Pretty dope, huh?

Note that since `Promise.coroutine()` returns a promise, any errors that occur within the coroutine function can be caught by appending a single `.catch()` to the function invocation. With a single line of code, we've error handled the whole thing!

{% highlight javascript linenos %}
  var Promise = require('bluebird');

  var ajaxRoutine = Promise.coroutine(function*(val) {
    var result1 = yield promisifiedAjaxCall();
    if (result1 === someValueIDontLike) return null;
    var result2 = yield anotherPromisifiedAjaxCall();
    if (result1 === someValue && result2 === someOtherValue) {
      doSomeThing();
    }
    else {
      doSomeOtherThing();
    }
  });

  /* ------ capturing return value ------ */
  var returnedVal = ajaxRoutine(val).catch(function(err) {
    if (err) throw err;
  });
{% endhighlight %}

But wait, there's more! `Promise.coroutine` actually returns a promise, so we can chain `.then` onto the coroutine invocation, like so:

{% highlight javascript linenos %}
  var Promise = require('bluebird');

  var ajaxRoutine = Promise.coroutine(function*(val) {
    var result1 = yield promisifiedAjaxCall();
    if (result1 === someValueIDontLike) return null;
    var result2 = yield anotherPromisifiedAjaxCall();
    if (result1 === someValue && result2 === someOtherValue) {
      doSomeThing();
    }
    else {
      doSomeOtherThing();
    }
  });

  /* ------ capturing return value ------ */
  var returnedVal = ajaxRoutine(val).then(function() {
    console.log('This won't happen until the coroutine is done executing!');
  }).catch(function(err) {
    if (err) throw err;
  });
{% endhighlight %}

Even more interestingly, since a generator is a function, we can actually use `return` statements! The only requirement is that we return a promise:

{% highlight javascript linenos %}
  var Promise = require('bluebird');

  var ajaxRoutine = Promise.coroutine(function*(val) {
    var result1 = yield promisifiedAjaxCall();
    if (result1 === someValueIDontLike) return null;
    var result2 = yield anotherPromisifiedAjaxCall();
    if (result1 === someValue && result2 === someOtherValue) {
      doSomeThing();
      return somePromise();
    }
    else {
      doSomeOtherThing();
      return someOtherPromise();
    }
  });

  /* ------ capturing return value ------ */
  var returnedVal = ajaxRoutine(val).then(function(resolvedPromiseValue) {
    console.log('This is the resolved value from the promise: ', resolvedPromiseValue);
  }).catch(function(err) {
    if (err) throw err;
  });
{% endhighlight %}

Hopefully now you understand how promises and generators can be combined to write elegant, easy-to-read asynchronous code. Spread the word! Next time someone complains to you about how they don't like Node.js because 'callback hell is unavoidable', tell them about generators and promises!

*Note: If you want to use generators in Node, you need to have version 0.12 or higher AND run node with the --harmony option enabled. For example: node --harmony server.js

Alternatively, you can use [io.js](https://iojs.org/en/index.html) which natively supports ES6 functionality.