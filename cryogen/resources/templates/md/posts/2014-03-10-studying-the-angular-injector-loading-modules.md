{:title "Studying the Angular JS Injector - loading modules"
 :layout :post
 :tags  ["javascript" "angularjs"]}

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

A module gets loaded with the following code:

```language-javascript
 function loadModules(modulesToLoad){
    var runBlocks = [], moduleFn, invokeQueue, i, ii;
    forEach(modulesToLoad, function(module) {
      if (loadedModules.get(module)) return;
      loadedModules.put(module, true);

      try {
        if (isString(module)) {
          moduleFn = angularModule(module);
          runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

          for(invokeQueue = moduleFn._invokeQueue, i = 0, ii = invokeQueue.length; i < ii; i++) {
            var invokeArgs = invokeQueue[i],
                provider = providerInjector.get(invokeArgs[0]);

            provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
          }
        } else if (isFunction(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else if (isArray(module)) {
            runBlocks.push(providerInjector.invoke(module));
        } else {
          assertArgFn(module, 'module');
        }
      } catch (e) {
        if (isArray(module)) {
          module = module[module.length - 1];
        }
        if (e.message && e.stack && e.stack.indexOf(e.message) == -1) {
          // Safari & FF's stack traces don't contain error.message content
          // unlike those of Chrome and IE
          // So if stack doesn't contain message, we create a new string that contains both.
          // Since error.stack is read-only in Safari, I'm overriding e and not e.stack here.
          /* jshint -W022 */
          e = e.message + '\n' + e.stack;
        }
        throw $injectorMinErr('modulerr', "Failed to instantiate module {0} due to:\n{1}",
                  module, e.stack || e.message || e);
      }
    });
    return runBlocks;
  }
```

The function returns an array of runBlocks - functions which are to be invoked after loading.

There are three ways to define a module in AngularJS. The first is by specifying a runBlock directly :

```language-javascript
angular.module(function($httpProvider) {
	console.log('Module is now running');
});
```

Or by passing an array :

```language-javascript
angular.module(['$httpProvider', function($httpProvider) {
	...
}]);
```

In both these cases the module is run with this line:

```language-javascript
	runBlocks.push(providerInjector.invoke(module));
```

The module function is invoked against the providerInjector. This means that we can inject the elusive `$provide` object into this function. This can then be used to register services directly.

```language-javascript
var injector = angular.injector([function($provide) {
	$provide.value('anInterestingFact', 'An ant has two stomachs. One for its own food and another for food to share');
}]);

injector.get('anInterestingFact');
// 'An ant has two stomachs. One for its own food and another for food to share'
```

Most likely the module will be defined with a string that identifies the name of the module. A module in Angular is typically setup as follows :

```language-javascript
angular.module('myModule', [dependency]);
```
If the module has been defined in this way, the following code is run:

```language-javascript
moduleFn = angularModule(module);
runBlocks = runBlocks.concat(loadModules(moduleFn.requires)).concat(moduleFn._runBlocks);

for(invokeQueue = moduleFn._invokeQueue, i = 0, ii = invokeQueue.length; i < ii; i++) {
	var invokeArgs = invokeQueue[i],
    	provider = providerInjector.get(invokeArgs[0]);

	provider[invokeArgs[1]].apply(provider, invokeArgs[2]);
}
```

First we retrieve the module object using the angularModule function. This function is defined outside of the injector and is just an alias for the angular.module function. When the module is setup two arrays are populated : `_runBlocks` and `_invokeQueue`. (The code that sets this module up is not within the injector module, so I won't be looking at this for the moment.)

### _runBlocks 

This gets populated with functions specified in the `angular.module('myModule').run` block. This code needs to be invoked as soon as the module is loaded. So we concat this array to our runBlocks return.

### _invokeQueue

The _invokeQueue is populated with each service that is added to the module using the familiar `angular.module('myModule').controller`, `angular.module('myModule').directive` et al. calls. Each item in the queue is an array with three elements. The first is the provider that will invoke the service, the second is the method on the provider to use and the third element is an array of any arguments passed to the service.

Lets look at an example to see how it all fits together. Say we setup our module as follows:

```language-javascript
angular.module('aNiceModule', [])
		.run(function() {
			console.log('running...');
		})
		.controller('aNiceController', function($scope) {
			console.log('setting up controller');
		});
```

We can call `angular.bootstrap(window.document.body, ['aNiceModule']);` to kick off the module loading. The `aNiceModule` module will have one entry in _runBlocks:

```language-javascript
function() {
	console.log('running...');
}
```

and one entry in `_invokeQueue`:

```language-javascript
['$controllerProvider', 'register', ['aNiceModule', function($scope) {...}]]
```

The `$controllerProvider` is the built-in Angular provider that enables registering controllers. Calling register will add this service to the list of available controllers. Note that nothing gets added to the injectors cache. This means controllers cannot be injected into a service. If you really needed to get access to a controller (you do when unit testing) you would inject the `$controller` provider and retreive the controller by calling `get(controllerName)`.

Lets try creating a factory service:

```language-javascript
angular.module('aNiceModule', [])
		.factory('aNiceFactory', function() {
			console.log('setting up factory');
			return {};
		});
```

When we load this module, `_invokeQueue` will have the following entry:

```language-javascript
['$provide', 'factory', ['aNiceModule', function() {...}]]
```

The service gets registered using the built in `$provideProvider`. The service is then available to the injector to inject into any other service that needs it.

That pretty much sums up the injector. The rest of Angular JS is built around this core module and it has proved immensely insightful to learn it inside out.

If you are interested in using it on the server in your nodeJS app, I have pulled out the injector code into [it's own module](https://github.com/FungusHumungus/pongular).