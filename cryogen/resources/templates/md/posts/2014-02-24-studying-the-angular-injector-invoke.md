{:title "Studying the Angular JS Injector - invoke"
 :layout :post
 :tags  ["javascript" "angularjs"]}

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

The invoke method invokes the given function with the parameters injected.

```language-javascript
function invoke(fn, self, locals){
      var args = [],
          $inject = annotate(fn),
          length, i,
          key;

      for(i = 0, length = $inject.length; i < length; i++) {
        key = $inject[i];
        if (typeof key !== 'string') {
          throw $injectorMinErr('itkn',
                  'Incorrect injection token! Expected service name as string, got {0}', key);
        }
        args.push(
          locals && locals.hasOwnProperty(key)
          ? locals[key]
          : getService(key)
        );
      }
      if (!fn.$inject) {
        // this means that we must be an array.
        fn = fn[length];
      }

      // http://jsperf.com/angularjs-invoke-apply-vs-switch
      // #5388
      return fn.apply(self, args);
    }
```

The first thing to note is that this method actually takes three parameters, the Angular documentation only mentions the first one.

Invoke first [annotates](http://taoofcode.net/studying-the-angular-injector-annotate/) the method to get a list of its parameters.

Each of these parameters are then validated to ensure they are a String. Then `invoke` locates the value to inject into the method with this code :

```language-javascript
 args.push(
          locals && locals.hasOwnProperty(key)
          ? locals[key]
          : getService(key)
        );
```

First, we look to see if the value is found in the `locals` parameter. [$routeProvider](http://docs.angularjs.org/api/ngRoute/provider/$routeProvider) uses this in `route.resolve` where you can setup additional dependencies to inject into the controller.

It is also possible to use this to override the registered service. Handy for unit testing.

If the value is not found in `locals`, `getService` is called to attempt to locate it.

Once all parameters are retrieved, the function is called with these parameters using [apply](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/apply) and its result is returned.


Next: [getService](http://taoofcode.net/studying-the-angular-injector-getservice)

