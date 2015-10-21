+++
title="Javascript Schönfinkeling"
date="2014-03-18"
tags=["javascript"]
+++

In Javascript is it extremely common to pass function as parameters to other functions.

```language-javascript
function addOne(a) {
	return a + 1;
}

[1,2,3,4].map(addOne);
```
This is all good. However when your function takes more than one parameter, you can't just pass the function directly so you need to create a new function which calls the original one:

```language-javascript
function add(a,b) {
	return a + b;
}

[1,2,3,4].map(function(a) {
	return add(a,1);
});
```


This does work, but you do lose some expressiveness by doing so - the semantics of the code gets lost in the syntax.

We can break up our add function to return a nested series of functions - one for each parameter:

```language-javascript
function add(a) {
	return function(b) {
		return a + b;
    };
}

[1,2,3,4].map(add(1));
```

This technique of taking a function that accepts multiple parameters and converting it into a chain of functions, each accepting one parameter, is known as [Currying](https://en.wikipedia.org/wiki/Currying) (also known as Schönfinkeling).

Although it is more expressive to call, it is a bit of a pain to have to write that curried function. It would be better if we could write something that could convert a standard function into a curried function.

So, instead of capturing the parameters in a series of nested closures, the following function will collect the passed parameter in an array. This is captured in a closure that is then returned for the next call in the chain. When all the required parameters have been collected, it calls the our function:

```language-javascript
var curry = function(fn) {

  var curryOrCall = function(args) {
    if (fn.length === args.length) {
      // All parameters have been passed, call the original function
      return fn.apply(this, args);
    }
    
    return function(param) {
      // Create a copy of the arguments
      var myargs=args.slice(0); 
      // Add the new argument to the list.
      myargs.push(param); 
      return curryOrCall(myargs);
    };
  };
  
  return curryOrCall([]); 
};
```

Used like:

```language-javascript
var add = curry(add(a,b) {
	return a + b;
});

[1,2,3,4].map(add(1));
```

This is all good and well, but we can only pass one parameter at a time, so it can get a little annoying, and inefficient, if there are several parameters you need to deal with. 


```language-javascript
var add3 = curry(add(a,b,c) {
	return a + b + c;
});

[1,2,3,4].map(add3(1)(2));
```

Pure currying means that each function in the chain **must** take only one parameter. In practical terms, often we will have multiple parameters that we want to pass, like: `[1,2,3,4].map(add3(1,2));`

###Partial function application

This is what *partial function application* is all about. Lets adjust our `curry` function to enable multiple parameters at a time:

```language-javascript
var partial = function(fn) {

  var partialOrCall = function(args) {
    if (fn.length === args.length) {
      // All parameters have been passed, call the original function
      return fn.apply(this, args);
    }

    return function(/* arguments */) {
      // Add all passed arguments to our arguments list
      var myargs=args.slice(0).concat(
      	Array.prototype.slice.call(arguments, 0));
      return partialOrCall(myargs);
    };
  };

  return partialOrCall([]);
}
```

So now it works with multiple parameters:


```language-javascript
var add = partial(add3(a,b,c) {
	return a + b + c;
});

[1,2,3,4].map(add3(1,2));

// Note currying still works if you really want to:
[1,2,3,4].map(add3(1)(2));

```
---
###Uses

####Passing to higher-order functions
Partial function application lets you be more expressive when passing functions to higher-order functions such as map.

```language-javascript
[1,2,3,4].map(function(a) {
	return add(4, a);
});
```

vs 

```language-javascript
[1,2,3,4].map(add(4));
```

---

####Creating member functions

If you have an object whose member functions invoke largely the same functionality but with some slight differences, you can set these members to be a partially applied function against one common function. This  is used in Angular JS:

```language-javascript
 function supportObject(delegate) {
    return function(key, value) {
      ...
    };
  }

providerCache = {
        $provide: {
            provider: supportObject(provider),
            factory: supportObject(factory),
            service: supportObject(service),
            value: supportObject(value),
            constant: supportObject(constant)
            ...
          }
      },
```

(This could be rewritten with our partial method as):

```language-javascript
var supportObject = partial(function(delegate, key, value) {
      ...
}

providerCache = {
        $provide: {
            provider: supportObject(provider),
            factory: supportObject(factory),
            service: supportObject(service),
            value: supportObject(value),
            constant: supportObject(constant)
            ...
          }
      },
```
