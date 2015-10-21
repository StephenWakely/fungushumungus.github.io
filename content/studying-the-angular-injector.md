+++
date="2014-02-19"
title="Studying the Angular JS Injector - intro"
tags=["javascript","angularjs"]
+++

I am truly impressed with the elegance of AngularJS and have been studying the source code to fully understand it. In this series I will be studying the Injector module as this is one of the core modules around which the rest of the framework revolves.

In [src/auto/injector.js](https://github.com/angular/angular.js/blob/481508d0e7ae9e4984ea380b9a43e589551c7a5b/src/auto/injector.js) there is a method `createInjector`. It is this method that kicks off the whole process.

After a bit of setup this method returns an `instanceInjector` object.

### instanceInjector
The injector is an object with the following functions:

```language-javascript
	return {
      invoke: invoke,
      instantiate: instantiate,
      get: getService,
      annotate: annotate,
      has: function(name) {
        return providerCache.hasOwnProperty(name + providerSuffix) || cache.hasOwnProperty(name);
      }
    };
```

First we go through these methods in turn.

* [annotate](http://taoofcode.net/studying-the-angular-injector-annotate)
* [invoke](http://taoofcode.net/studying-the-angular-injector-invoke)
* [get](http://taoofcode.net/studying-the-angular-injector-getservice)
* [instantiate](http://taoofcode.net/studying-the-angular-js-injector-instantiate)

Then we tie it all together by looking closer at the createInjector method to see how the injector is setup and used : [the twin injectors](studying-the-angular-injector-the-twin-injectors)

Finally we look into how a module is loaded and the services are registered with the injectors: [loadModules](http://taoofcode.net/studying-the-angular-injector-loading-modules)
