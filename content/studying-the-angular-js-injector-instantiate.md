+++
date="2014-02-24"
title="Studying the Angular JS Injector - instantiate"
tags=["javascript","angularjs"]
+++

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

Whilst `invoke` calls a function with it's parameters injected, `instantiate` will contruct a new object with it's constructor parameters injected.

`instantiate` gives us an excellent insight into how javascript objects work. In javascript, a class is just a function and an class instance is just a function that has been invoked with the `new` operator.

Say we have a simple class :

```language-javascript
function Person(firstName, lastName) {
	this.firstName = firstName;
    this.lastName = lastName;
}
```

We can add methods to this class via the functions `prototype` property.

```language-javascript
Person.prototype.beNiceTo = function() {
	console.log(this.firstName + ' ' + this.lastName + ' is my friend';
};
```

Then we can create an instance of this class with `new`.

```language-javascript
> var plum = new Person('Professor', 'Plonk');
> plum.beNiceTo();
Professor Plonk is my friend
```

All is good.

Now in order to invoke a method and inject parameters into it, the Injector takes advantage of the [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) method. This method allows you to invoke a function and pass in an array which will become that functions parameters. 

Very useful, but the problem with `apply` is that it only works when you want to call a function. There is no way to mix `new` with `apply`. The Injector gets around this problem in a pretty clever way.

Lets look at the code.

```language-javascript
function instantiate(Type, locals) {
      var Constructor = function() {},
          instance, returnedValue;

      // Check if Type is annotated and use just the given function at n-1 as parameter
      // e.g. someModule.factory('greeter', ['$window', function(renamed$window) {}]);
      Constructor.prototype = (isArray(Type) ? Type[Type.length - 1] : Type).prototype;
      instance = new Constructor();
      returnedValue = invoke(Type, instance, locals);

      return isObject(returnedValue) || isFunction(returnedValue) ? returnedValue : instance;
    }
```

First we create a new, completely empty, object called `Constructor`.

If we had passed the above Person class into this method (into the Type parameter) we would then have these two classes:

```
+--------------+ +---------------+
|  beNiceTo    | |               |
+--------------+ +---------------+
       ^                 ^
       |                 |
       |    Prototype    |
       |                 |
+------+-------+ +-------+-------+
|  Type        | | Constructor   |
|--------------| |---------------|
|              | |               |
|              | |               |
+--------------+ +---------------+
```

Next, we set our Constructors prototype to this new Constructors prototype. Copying methods between class prototypes is completely legal to do in javascript - it really is a very malleable language. Note, if Type is an array, this means that it is actually an annotated array - `['$scope', function($scope) { }]` and the classes constructor function is actually the last element of the array.

We now have these two classes :

```
+--------------+
|  beNiceTo    | +-------+
+--------------+         |
       ^                 |
       |                 |
       |    Prototype    |
       |                 |
+------+-------+ +-------+-------+
|  Type        | | Constructor   |
|--------------| |---------------|
|              | |               |
|              | |               |
+--------------+ +---------------+
```

An instance of this class is then created. This doesn't do too much yet. It creates an object with our Types instance methods, but it hasn't actually invoked the constructor yet.

```
+--------------+
|  beNiceTo    | +-------+
+--------------+         |
       ^                 |
       |                 |
       |    Prototype    |
       |                 |
+------+-------+ +-------+-------+
|  Type        | | Constructor   |
|--------------| |---------------|
|              | |               |
|              | |               |
+--------------+ +---------------+
                         ^
                         |
                         |
                 +-------+-------+
                 |   Instance    |
                 |---------------|
                 |               |
                 |               |
                 +---------------+
```

We still need to invoke this constructor and inject the parameters into it. To do this we call `invoke` with our type as normal. 

There is one significant difference - we pass the new Constructor instance that we have just created. When `invoke` calls the method with the injected parameters it does this using `fn.apply(self, args)`. Looking back at the [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) documentation, the first parameter sent will become the `this` variable in the function. So, by invoking with our newly created class, the constructor is called against it.

So we have managed to create our new class without ever actually calling `new` on it.

```
+--------------+
|  beNiceTo    | +-------+
+--------------+         |
       ^                 |
       |                 |
       |    Prototype    |
       |                 |
+------+-------+ +-------+-------+
|  Type        | | Constructor   |
|--------------| |---------------|
|              | |               |
|              | |               |
+--------------+ +---------------+
                         ^
                         |
                         |
                 +-------+-------+
                 |   Instance    |
                 |---------------|
                 | Professor     |
                 | Plonk         |
                 +---------------+
```

This instance with injected parameters is then returned.

Next, we will tie everything together by looking at how the Injector is set up: [the twin injectors](http://taoofcode.net/studying-the-angular-injector-the-twin-injectors)