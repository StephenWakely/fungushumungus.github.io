+++
title="Studying the Angular JS Injector - getService"
date="2014-02-24"
tags=["javascript","angularjs"]
+++

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

The `getService` function is the work-horse of `invoke`. This is the method that takes a service name and attempts to locate it in the list of registered services.

```language-javascript
	function getService(serviceName) {
      if (cache.hasOwnProperty(serviceName)) {
        if (cache[serviceName] === INSTANTIATING) {
          throw $injectorMinErr('cdep', 'Circular dependency found: {0}', path.join(' <- '));
        }
        return cache[serviceName];
      } else {
        try {
          path.unshift(serviceName);
          cache[serviceName] = INSTANTIATING;
          return cache[serviceName] = factory(serviceName);
        } catch (err) {
          if (cache[serviceName] === INSTANTIATING) {
            delete cache[serviceName];
          }
          throw err;
        } finally {
          path.shift();
        }
      }
    }
```
When the injector is created it is passed two parameters, `cache` and `factory`. These are critical and we will break these down soon. For now the cache is a list of services that have been instantiated and factory is a method that instantiates a service given it's name.

If the service already exists in the cache, it looks to see if the service is actually the `INSTANTIATING` placeholder. If it is, this means that whilst instantiating this service it has invoked another service that is then trying to instantiate this one again - a circular dependency. The injector can't deal with this so it throws an error. Otherwise the service is just returned.

If the service is not in the cache it needs to be instantiated.

```language-javascript
	path.unshift(serviceName);
    cache[serviceName] = INSTANTIATING;
    return cache[serviceName] = factory(serviceName);
```

The service name is added to the beginning of the `path` array. This is just an array used to keep track of the route taken through services as the injector instantiates them. It is used to report useful error messages when something goes wrong.

The entry in the cache for this service is created and first gets set to this INSTANTIATING placeholder. The `factory` function is then called to created our service, its result is stored in the cache and the value is returned.

Next: [instantiate](http://taoofcode.net/studying-the-angular-js-injector-instantiate)