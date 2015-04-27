---
layout: post
title:  "Javascript Hidden Classes and Inline Caching in V8"
date:   2015-04-26 14:04:21
categories: jekyll update
---
Hidden Classes
==============
Javascript is a dynamic programming language which means that properties can easily be added or removed from an object after its instantiation. For example, in the code snippet below an object is instantiated with the properties "make" and "model"; however, after the object has already been created, the "year" property is dynamically added.


{% highlight javascript %}

var car = function(make,model) {
	this.make = make;
	this.model = model;
}

var myCar = new car(honda,accord);

myCar.year = 2005;

{% endhighlight %}

Most Javascript interpreters use dictionary-like objects ([hash function](http://en.wikipedia.org/wiki/Hash_function) based) to store the location of object property values in memory. This structure makes retrieving the value of a property in Javascript more computationally expensive than it would be in a non-dynamic programming language like Java. In Java, all of an objects properties are determined by a fixed object layout before compilation and cannot be dynamically added/removed at runtime. As a result, the values of properties (or pointers to those properties) can be stored as a contiguous buffer in memory with a fixed-offset between each one. The length of an offset can easily be determined based on the properties type, whereas this is not possible in Javascript where a properties type can change during runtime. [This short blog post](http://www.programcreek.com/2011/11/what-do-java-objects-look-like-in-memory/) does a good job of explaining what Java objects look like in memory.

In a non-dynamic language like Java, a properties location in memory can often be determined with only a single instruction whereas in Javascript several instructions are required to retrieve the location from a hash table. As a result, property lookup is much slower in Javascript than it is in other languages.

Since the use of dictionaries to find the location of object properties in memory is so inefficient, V8 uses a different method instead: hidden classes. Hidden classes work similarly to the fixed object layouts (classes) used in languages like Java, except they are created at runtime. While reading the rest of this post, keep in mind that V8 attaches a hidden class to each and every object, and the purpose of the hidden classes is to optimize property access time. Now, Lets take a look at what they actually look like.

{% highlight javascript %}
function point(x,y) {
	this.x = x;
	this.y = y;
}

var obj = new point(1,2);

{% endhighlight %}


Once the new function is declared, Javascript will create hidden class C0. 

![Source: developers.google.com]({{ site.url }}/assets/hidden-classes/hiddenClass1.png)

No properties have been defined for point yet, so C0 is empty.

Once the first statement "this.x = x" is executed, V8 will create a second hidden class called C1 that is based on C0. C1 describes the location in memory (relative to the object pointer) where the property x can be found. In this case, x is stored at [offset](http://en.wikipedia.org/wiki/Offset_%28computer_science%29) 0, which means that when viewing a point object in memory as a contigous buffer, the first offset will correspond to property x. V8 will also update C0 with a "class transition" which states that if a property x is added to a point object, the hidden class should switch to C1, and that is exactly what happens; the hidden class for the point object below is now C1.

![Source: developers.google.com]({{ site.url }}/assets/hidden-classes/hiddenClass2.png)

> Everytime a new property is added to an object, the objects old hidden class is updated with a transition path to the new hidden class. Hidden class transitions are important because they allow hidden classes to be shared among objects that are created in the same way. If two objects share a hidden  class and the same property is added to both of them, transitions ensure that both objects receive the same new hidden class and all the optimized code that comes with it.

This process is repeated when the statement "this.y = y" is executed. A new hidden class called C2 is created, a class transition is added to C1 stating that if a property "y" is added to a point object (that already contains property "x") then the hidden class should change to C2, and the point objects hidden class is updated to C2.

![Source: developers.google.com]({{ site.url }}/assets/hidden-classes/hiddenClass3.png)

Note: Hidden class transitions are dependent on the order in which properties are added to an object. Take a look at the code snippet below:

{% highlight javascript %}

1  function point(x,y) {
2    this.x = x;
3    this.y = y;
4  }
5 
7  var obj1 = new point(1,2);
8  var obj2 = new point(3,4);
9
10 obj1.a = 5;
11 obj1.b = 10;
12
13 obj2.b = 10;
14 obj2.a = 5;

{% endhighlight %}

Up until line 9, obj1 and obj2 shared the same hidden class. However, since properties a and b were added in opposite orders, obj1 and obj2 end up with different hidden classes as a result of following separate transition paths.

![]({{ site.url }}/assets/hidden-classes/propertyAddOrder.jpg)

If you've been following along closely, your first instinct might be to think that obj1 and obj2 having two different hidden classes isn't a big deal. As long as each of their hidden classes stores the appropriate offsets, accessing their properties should be just as fast as if they shared a hidden class right? In order to understand why this isn't true, we need to take a look at another optimization technique employed by V8 called inline caching.

Inline Caching
==============
V8 takes advantage of another commonly used technique for optimizing dynamically typed languages called "inline caching". An in-depth explanation of inline caches in Javascript can be found [here](https://github.com/sq/JSIL/wiki/Optimizing-dynamic-JavaScript-with-inline-caches), but in simple terms inline caching relies upon the observation that repeated calls to the same method tend to occur on the same type of object.

So how does it work? V8 maintains a cache of the type of objects that were passed as a parameter in recent method calls, and uses that information to make an assumption about the type of object that will be passed as a parameter in the future. If V8 is able to make a good assumption about the type of object that will be passed to a method, it can bypass the process of figuring out how to access the objects properties, and instead use the stored information from previous lookups to the objects hidden class.

So how are the concepts of hidden classes and inline caching related? Whenever a method is called on a specific object, the V8 engine has to perform a lookup to that objects hidden class to determine the offset for accessing a specific property. After two successful calls of the same method to the same hidden class, V8 omits the hidden class lookup and simply adds the offset of the property to the object pointer itself. For all future calls of that method, the V8 engine _assumes_ that the hidden class hasn't changed, and jumps directly into the memory address for a specific property using the offsets stored from previous lookups; this greatly increases execution speed. 

Inline caching is also why its so important that objects of same type share hidden classes. If you create two objects of the same type, but with different hidden classes (as we did in the example earlier), V8 won't be able to use inline caching because even though the two objects are of the same type, their corresponding hidden classes assign different offsets to their properties.

![]({{ site.url }}/assets/hidden-classes/diffHiddenClasses.jpg)

Of course, Javascript being a dynamically typed language, ocassionally the assumption about the hidden class of the object will be incorrect, and in that case V8 will "de-optimize" and revert back to the original version of the method call in which the objects hidden class is checked.

Optimization takeaways
======================

1. Always instantiate your object properties in the same order so that hidden classes, and subsequently optimized code, can be shared.
2. Adding properties to an object after instantiation will force a hidden class change and slow down an methods that were optimized for the previous hidden class. Instead, assign all of an objects properties in its constructor.
3. Code that executes the same method repeatedly will run faster than code that executes many different methods only once (due to inline caching).

Sources
=======
+ [https://developers.google.com/v8/design?hl=sv&csw=1#prop_access](https://developers.google.com/v8/design?hl=sv&csw=1#prop_access)
+ [https://developers.google.com/v8/videos?hl=sv#video0a](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [http://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [https://medium.com/@twokul/hidden-classes-in-javascript-and-inline-caching-a02940939e25](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [http://en.wikipedia.org/wiki/Inline_caching](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [http://en.wikipedia.org/wiki/Function_overloading](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [https://developers.google.com/v8/videos?hl=sv#video0ahttps://livoris.net/node/65](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [http://www.programcreek.com/2011/11/what-do-java-objects-look-like-in-memory/](https://developers.google.com/v8/videos?hl=sv#video0a)
+ [https://github.com/sq/JSIL/wiki/Optimizing-dynamic-JavaScript-with-inline-caches](https://developers.google.com/v8/videos?hl=sv#video0a)