{:title "Creating a plugin system in Angular JS with the $compile service"
 :layout :post
 :tags  ["javascript"]}

Angular JS directives are powerful. Using them allows you to manipulate pretty much everything in the DOM that you would want to. But there is one exception. Dynamically creating a directive depending on data received from the server, something often used for plugin systems. Luckily we can access the $compileProvider directly to work around these limitations.

##Plugins
Say you are designing a plugin system. Each plugin is implemented as a different directive. The plugin to use is stored in the database and sent down from the server.

Lets set up a couple of very simple directives:

```language-javascript
angular.module('anApp').directive('one', function() {
    return {
        template: '<span>I am one</span>'   
    };
});

angular.module('anApp').directive('two', function() {
    return {
        template: '<span>I am two</span>'   
    };
});
```

Our controller grabs the plugin to use from a factory and attaches it to the scope. This factory has presumably queried the server to get the name of the plugin we need :

```language-javascript
angular.module('anApp').controller('aCtrl', function($scope, aFactory) {
    aFactory().then(function(plugin) {
       $scope.plugin = plugin; // Either 'one' or 'two'.
    });
})
```

Now we have a problem. How are we going to render the required directive?

###Naive approaches

A naive approach might be something like :

```language-javascript
<div ng-app='anApp'>
    <div ng-controller='aCtrl'>
        <div {{plugin}}></div>
    </div>
</div>
```

That doesn't work. The `{{plugin}}` does not get interpolated in time and so the directive is not rendered.

Another approach that could almost work is specifying a directive that uses a function to specify the template.

```language-javascript
.directive('taoPlugin', function() {
    return {
        template: function(element, attrs) { 
            var p = attrs.plugin;
            return '<' + p + ' />'; 
        }  
    }   
});
```

This could almost work. The problem is the attrs doesn't contain the correct plugin name. We want to pull the plugin name out of our parents controllers scope. There is no way of pulling the actual data out of the scope into the attrs collection. (Feel free to correct me if I am wrong - it would be great if there was!) 

So we just end up with the following output rendered to page :

```language-javascript
<one />
```
---

###$compile to the rescue.

$compile is the provider that takes an HTML template string and creates a template function. When this template function is called with a `$scope` it spits out HTML. It is the provider at the core of Angular which enables directives to manipulate the DOM using the Angular templating language.

It takes template HTML : 

```language-javascript
<div>{{something}}</div>
``` 

and returns a function, which we call with a scope :

```language-javascript
$scope.something = 'Interesting text'
``` 

and compiles it to : 

```language-javascript
<div>Interesting text</div>
```
(Note the text is not acually interpolated until it is rendered to the DOM.)

So, if we pass `$compile` the string : `'<one />'` it is actually going to compile our `one` directive. Naturally because we are just passing a string, we will have no problems building this string up at run time to whatever we require.

Then all we need to do is append this html to a given element for it to be rendered on the page. To access an element on the page we need to create a directive. So lets create this directive.

```language-javascript
angular.module('anApp').directive('taoPlugin', ['$compile', function($compile) {
    return {
        restrict: 'E',
        scope: { 'plugin': '=' },
        link: function(scope, element) {
            var template = '<' + scope.plugin + ' />',
                compiled = $compile(template)(scope);

			element.append(compiled);            
        }   
    }   
}]);
```

We can then setup our directive simply :

```language-javascript
<tao-plugin plugin='plugin'></tao-plugin>
```

The directive we have set in our controller  : `$scope.plugin = plugin;` is the directive that is rendered.

Here is a [jsFiddle](http://jsfiddle.net/ht8ZQ/26/) that demonstrates this.

Note our directive here isn't passing on any parameters to the plugin directive. This could be achieved by looping round the attr array that gets passed into the link function.

Also note that the directive has isolate scope. This gets passed on to the plugin directives. If you want a different scope for your plugins, you would need to change the scope of this plugin as well.