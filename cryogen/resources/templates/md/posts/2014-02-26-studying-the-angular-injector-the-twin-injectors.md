{:title "Studying the Angular JS Injector - the twin injectors"
 :layout :post
 :tags  ["javascript" "angularjs"]}

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

When Angular creates the injector, it actually creates two injectors:

```language-javascript
providerInjector = (providerCache.$injector =
	createInternalInjector(providerCache, function() {
		throw $injectorMinErr('unpr', "Unknown provider: {0}", path.join(' <- '));
	})),
          
instanceInjector = (instanceCache.$injector =
	createInternalInjector(instanceCache, function(servicename) {
		var provider = providerInjector.get(servicename + providerSuffix);
		return instanceInjector.invoke(provider.$get, provider);
	}));
```


Two parameters are passed to the createInternalInjector function. The first is the cache to use to look up instances (a simple object). The second is a factory function. The factory function is used to create a service when it doesn't exist in the cache.

###the instanceInjector

The instanceInjector is the injector that is returned when you call createInjector.

The instanceInjector stores the list of instantiated services in the system. It is initialised with an empty object. The providerInjector maintains the list of uninstantiated services. 

Looking at the factory function for instance injector, when we are trying to fetch a service that has not yet been instantiated, we will look up the service name in the providerInjector with the name `servicename + providerSuffix`. providerSuffix is the string `Provider`. When we have this we invoke the `$get` function of the provider object.

There are two assumptions made here. 

1. All services stored in the `providerInjector` are named with a suffix `Provider`.
2. All services stored in the `providerInjector` are objects with a `$get` function.

---
###the providerInjector

Lets see how the providerInjector is set up.

The cache for the providerInjector is initialised with one service - $provide :

```language-javascript
 providerCache = {
        $provide: {
            provider: supportObject(provider),
            factory: supportObject(factory),
            service: supportObject(service),
            value: supportObject(value),
            constant: supportObject(constant),
            decorator: decorator
          }
      }
```
The $provide service is always available on the providerInjector by default. It is through this service that all other services are registered.

supportObject is the following method:

```language-javascript
function supportObject(delegate) {
    return function(key, value) {
      if (isObject(key)) {
        forEach(key, reverseParams(delegate));
      } else {
        return delegate(key, value);
      }
    };
  }
```
This is a very common javascript pattern. A function that captures a variable (in this case `delegate`) that then returns another function. 

The returned function from `supportObject` will apply the delegate to the parameters, or if an object is passed in apply the delegate to all the fields of that object.

The delegate passed in is a function that handles the creation of either a provider, factory, service, value or constant. Each of these work slightly differently, but they all end up adding a service to the providerCache named with a `Provider` suffix and having a `$get` member function.

####provider
```language-javascript
  function provider(name, provider_) {
    assertNotHasOwnProperty(name, 'service');
    if (isFunction(provider_) || isArray(provider_)) {
      provider_ = providerInjector.instantiate(provider_);
    }
    if (!provider_.$get) {
      throw $injectorMinErr('pget', "Provider '{0}' must define $get factory method.", name);
    }
    return providerCache[name + providerSuffix] = provider_;
  }
```

If the given provider is a function or an array, that provider is invoked. Then we validate to ensure a $get property is included in the provider and the object is added to the `providerCache`.

####factory

```language-javascript
  function factory(name, factoryFn) { return provider(name, { $get: factoryFn }); }
```

A factory just creates a provider with a $get property that points to the passed in factory function.

####service
```language-javascript
function service(name, constructor) {
    return factory(name, ['$injector', function($injector) {
      return $injector.instantiate(constructor);
    }]);
  }
```

A service creates a factory. The method is annotated to retrieve the `$injector` object. The injector is used to instantiate the an instance of the constructor class that is passed to the service.

####value
```language-javascript
function value(name, val) { return factory(name, valueFn(val)); }
```

A value creates a factory with a function that returns the value.

####constant
```language-javascript
 function constant(name, value) {
    assertNotHasOwnProperty(name, 'constant');
    providerCache[name] = value;
    instanceCache[name] = value;
  }
```
Constants break the $get and provider suffix rules. The value just gets set directly in both the provider and instance caches as there is no processing that needs doing to instantiate a constant.

---

So why would they implement this twin injector solution? Couldn't they mix all the instantiated and uninstantiated services into one injector? Their names are already distinguished by the `provider` suffix on the uninstantiated services.

One reason is hiding functionality. Supposing you wanted to register a service with the injector directly. If you tried you would get the following:

```language-javascript
> var injector = angular.injector()
> injector.get('$provide').value('myValue', 3.14);
Error: [$injector:unpr] Unknown provider: $providerProvider <- $provider
```

You only have access to the instanceInjector. When this does a lookup on the providerInjector, it adds the Provider suffix to the name. It attempts to lookup $providerProvider, which doesn't exist.

So if we can't access $provider how can we register our services with the Injector?

This is all done via modules. All services set up in angular must be attached to a particular module and this is the only way to register the service with the injector.

Next: [loadModules](http://taoofcode.net/studying-the-angular-injector-loading-modules)


