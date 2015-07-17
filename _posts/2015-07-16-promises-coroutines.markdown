---
layout: post
title:  "Combining Promises and Coroutines to write Elegant Asynchronous code in Javascript"
date:   2015-04-26 14:04:21
categories: jekyll update
---

Introduction
==============

The purpose of this blog post is to teach you how promises and coroutines can be combined to write asynchronous code in Javascript that *looks* synchronous. Writing code this way makes it very easy to read and reason about what the code is doing. I'm not going to cover the topic of promises because there are tons of resources online that provide much better explanations than I could. I will, however, provide a brief overview of coroutines in Javascript, but I recommend that you pursue additional reading elsewhere.

Coroutines
==========

Coroutines were added to Javascript as a result of ES6. A coroutine in JavaScript is a special type of function that can halt itself at any time using the keyword yield, and then later resume execution. Whats so interesting about coroutines is that it allows a developer to write a function that not only can "return" multiple times, but can also have values "injected" into it mid-execution.

If that sounded confusing, don't worry, coroutines quickly begin to make sense once you start playing around with them. That said, open up your favorite ES6-compliant* Javascript REPL and paste in the following code:

{% highlight javascript linenos %}
  /*the asterisk indicates that the function is a 
  coroutine instead of a traditional function */
  var coroutineFactory = function* () {
    var result1 = yield 1;
    console.log(result1);
    var result2 = yield 2;
    console.log(result2);
    var result3 = yield 3;
    console.log(result3);
  }

  var testCoroutine = new coroutineFactory();
{% endhighlight %}

Now try calling testCoroutine.next() a few times. You should get something that looks like this:

{% highlight javascript linenos %}
console.log(testCoroutine.next()); 
// Object {value: 1, done: false}
console.log(testCoroutine.next()); 
// undefined
// Object {value: 2, done: false}
console.log(testCoroutine.next());
// undefined
// Object {value: 3, done: false}
console.log(testCoroutine.next());
// undefined
// Object {value: undefined, done: true}
{% endhighlight %}

Everytime you call testCoroutine.next(), the function executes until it encounters a yield statement. The yield statement functions like "return" in that it returns whatever value you yield. So the first time you call testCoroutine.next() the coroutineFactory function will execute until it encounters the first yield statement:

{% highlight javascript %}
  var result1 = yield 1;
{% endhighlight %}

At this point the coroutine will yield (pausing execution) and the statement testCoroutine.next() will evaluate to an object that contains two properties:
  
1. value: the yielded value
2. done: a boolean value that represents whether or not the coroutine has any more code to execute (false in this case) 

The coroutine will remain paused until the next time testCoroutine.next() is called. This process repeats itself everytime we call testCoroutine.next() until there are no more yield statements, at which point the done property will be true.

But why did all the console logs evaluate to undefined? When you call .next(), the yield statement is replaced with whatever parameters you pass to .next(). This is how values can be "injected" into the coroutines mid-execution. To see this in action, create a new testCoroutine and then execute the following code:

{% highlight javascript linenos %}
console.log(testCoroutine.next()); 
// Object {value: 1, done: false}
console.log(testCoroutine.next(4)); 
// 4
// Object {value: 2, done: false}
console.log(testCoroutine.next(5));
// 5
// Object {value: 3, done: false}
console.log(testCoroutine.next(6));
// 6
// Object {value: undefined, done: true}
{% endhighlight %}

The values that are passed into .next() as parameters are stored in the result variables and subsequently console logged.

Combining Coroutines and Promises
=================================

Ok, now that we have a basic understand of promises and coroutines, how can we combine these to write elegant and easy-to-understand asychronous code? Well first, lets take a look at the problem we're trying to solve:

{% highlight javascript linenos %}
  var ajaxRoutine = function(val) {
    var result1 = yield promisifiedAjaxCall(val).then(function(result1) {
      var result2 = yield anotherPromisifiedAjaxCall().then(function(result2) {
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

Even with only two asynchronous calls, this function is starting to get a little messy, and we're not even handling errors yet! Lets try rewriting this function using a combination of promises and a coroutines:

{% highlight javascript linenos %}
  var ajaxRoutine = function*(val) {
    var result1 = yield promisifiedAjaxCall(val);
    var result2 = yield anotherPromisifiedAjaxCall();
    if (result1 === someValue && result2 === someOtherValue) {
      doSomeThing();
    }
    else {
      doSomeOtherThing();
    }
  }
  var testCoroutine = new ajaxRoutine(val);
  var result1Promise = testCoroutine.next().then(function(result) {
    testCoroutine.next(result);
  });
  var result2Promise = testCoroutinen.next().then(function(result) {
    testCoroutine.next(result);
  });
{% endhighlight %}

So whats going on here? Lets walk through how this code will execute. The first time testCoroutine.next() is called on line 12 the ajaxRoutine coroutine will run until it hits the first yield statement on line 2.

{% highlight javascript %}
  var result1 = yield promisifiedAjaxCall(val);
{% endhighlight %}

result1Promise variable will then be assigned a promise. Once the ajax call is complete and the promise resolves, the .then() function will execute and response from the ajax call is passed back into the ajaxRoutine function and assigned to result1.

{% highlight javascript %}
  var result1Promise = testCoroutine.next().then(function(result) {
    testCoroutine.next(result);
  });
{% endhighlight %}

The code will continue to execute until it hits the yield statement on line 3 at which point the process will repeat, eventually assigning the result of the second ajax call to result2. The code can now be thought of as looking like this:

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

Now that the ajax calls have been resolved, its very easy to reason about the remaining code in linear fashion as you would with any synchronous functon. This is nice, however, actually invoking the function and feeding the resolved promises back into the coroutine is a messy process. What we want is something that not only can yield promises, but can automatically wait for them to resolve and then resume execution. Luckily, the Bluebird promise library has such a function. The code snippet above can be rewritten like this:

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

This time, the same process will occur and the code will execute asynchronously, waiting for each ajax request to complete before continuing, but the code can be read and reasoned about in a synchronous manner. Pretty sweet, huh? Note that since Promise.coroutine() returns a promise, any errors that occur within the coroutine function can be caught by appending a single .catch() to the function invocation. On top of that, we've regained the ability to use return statements! We could easily do something like this:

{% highlight javascript linenos %}
  var Promise = require('bluebird');
  var ajaxRoutine = Promise.coroutine(function*(val) {
    var result1 = yield promisifiedAjaxCall();
    if (result1 === someValueIDontLike) return null;
    var result2 = yield anotherPromisifiedAjaxCall();
    if (result1 === someValue && result2 === someOtherValue) {
      doSomeThing();
      return someThing;
    }
    else {
      doSomeOtherThing();
      return SomeOtherThing;
    }
  });

  /* ------ capturing return value ------ */
  var returnedVal = ajaxRoutine(val).catch(function(err) {
    if (err) throw err;
  });
{% endhighlight %}

Hopefully now you understand how promises and coroutines can be combined to write elegant, easy-to-read asynchronous code. Spread the word! Next time someone complains to you about how they don't like Node.js because 'callback hell is unavoidable', tell them about coroutines and promises!

*Note: If you want to use coroutines in Node, you need to have version 0.12 or higher AND run node with the --harmony option enabled. For example: node --harmony server.js