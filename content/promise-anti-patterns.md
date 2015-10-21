+++
date="2014-02-12"
title="Promise Anti-patterns"
tags=["javascript"]
+++

Promises are very simple once you get your head around them, but there are a few gotchas that can leave you with your head scratching. Here are a few that got me.

---

##  Nested Promises

You get a whole bundle of promises nested in eachother:

```language-javascript
loadSomething().then(function(something) {
	loadAnotherthing().then(function(another) {
                	DoSomethingOnThem(something, another);
	});
});
```
The reason you've done this is because you need to do something with the results of both promises, so you can't chain them since the `then()` is only passed the result of the previous return.

The real reason you've done this is because you don't know about the `all()` method.

To fix :

```language-javascript
q.all([loadSomething(), loadAnotherThing()])
	.spread(function(something, another) {
		DoSomethingOnThem(something, another);
});
```
Much simpler. The promise returned by `q.all` will resolve with an array of the results that is passed to `then()`. The `spread()` method will split this array up amongst the parameters.

---

## The Broken Chain

You have code like :

```language-javascript
function anAsyncCall() {
	var promise = doSomethingAsync();
	promise.then(function() {
    	somethingComplicated();
    });
    
	return promise;
}
```
The problem here is that any error raised in the `somethingComplicated()` method will not get caught. Promises are meant to be chained - each call to `then()` returns a new promise. It is this promise that the next `then()` should be called against. Generally the final call will be a `catch()`, any errors that are raised anywhere in the chain will be handled here.

In the above code the chain gets broken when you return the first promise rather than the result of the final `then()`.

The fix:

```language-javascript
function anAsyncCall() {
	var promise = doSomethingAsync();
	return promise.then(function() {
    	somethingComplicated()
    });   
}
```
Always return the result of the final `then()`.


---

## The Collection Kerfuffle

You have an array of items and you want to do something asynchronous on each of them. So you find yourself doing something involving recursion.

```language-javascript
function workMyCollection(arr) {
	var resultArr = [];
	function _recursive(idx) {
		if (idx >= resultArr.length) return resultArr;
            
		return doSomethingAsync(arr[idx]).then(function(res) {
			resultArr.push(res);
			return _recursive(idx + 1);
		});
	}

	return _recursive(0);
}
```

Ug, the code is hardly intuitive. The problem is that chaining methods when you don't know how many to chain can be painful. That is until you remember `map()` and `reduce()`.

To fix:

Remember `q.all` takes an array of promises and resolves to an array of the results. We can simply map the async calls to their results as follows :

```language-javascript
function workMyCollection(arr) {
	return q.all(arr.map(function(item) {
		return doSomethingAsync(item);
	}));    
}
```

Unlike the recursive non-solution, this will kick off all the async calls in parallel. Obviously much more time efficient.

If you need to run your promises in series, you can use reduce.

```language-javascript
function workMyCollection(arr) {
	return arr.reduce(function(promise, item) {
		return promise.then(function(result) {
			return doSomethingAsyncWithResult(item, result);
		});        
	}, q());
}
```

Not quite as tidy, but certainly tidier.

---

## The Ghost Promise

A certain method needs to do something asynchronous and sometimes it doesn't. So you create a promise even when you don't need one just to keep the two code paths consistent.

```language-javascript
var promise;
if (asyncCallNeeded) 
	promise = doSomethingAsync();
else
	promise = Q.resolve(42);
        
promise.then(function() {
	doSomethingCool();
});
```

Not the worst anti-pattern here, but there is definitely a cleaner way - wrap the 'value or promise' with `Q()`. This method will take either a value or a promise and act accordingly.

```language-javascript
Q(asyncCallNeeded ? doSomethingAsync() : 42)
	.then(
		function(value){
			doSomethingGood();
		})
    .catch( 
		function(err) {
			handleTheError();
		});
```

*Note: I was originally advising using Q.when there. Thankfully Kris Kowal has put me straight in the comments below. Don't use Q.when, just use Q() - it is much cleaner.*

---

## The Overly Keen Error Handler

The `then()` method takes two parameters, the fulfilled handler and the rejected handler. You might be tempted to do something like this:

```language-javascript
somethingAsync.then(
	function() {
		return somethingElseAsync();
	},
	function(err) {
		handleMyError(err);
});
```

The problem with this is that any errors occurring in the fulfilled handler does not get passed to the error handler.

To fix, make sure the error handler is in a separate then :
    
```language-javascript
somethingAsync
	.then(function() {
    	return somethingElseAsync();
	})
    .then(null,
		function(err) {
			handleMyError(err);
		});
```

Or use `catch()`:

```language-javascript
somethingAsync
	.then(function() {
		return somethingElseAsync();
	})
	.catch(function(err) {
		handleMyError(err);
	});
```
This ensures any errors occuring in the chain will get handled.

---

## The Forgotten Promise
You call a method that returns a promise. However, you forget about this promise and create your own one.

```language-javascript
var deferred = Q.defer();
doSomethingAsync().then(function(res) {
	res = manipulateMeInSomeWay(res);
	deferred.resolve(res);
}, function(err) {
	deferred.reject(err);
});
    
return deferred.promise;
```

This really is defeating the point of promises - simplicity. A lot of pointless code here.

To fix, just return the promise.

```language-javascript
return doSomethingAsync().then(function(res) {
	return manipulateMeInSomeWay(res);
});
```
