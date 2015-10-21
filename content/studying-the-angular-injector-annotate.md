+++
title="Studying the Angular JS Injector - annotate"
date="2014-02-19"
tags=["javascript","angularjs"]
+++

(This post is part of a series [studying the AngularJS injector](http://taoofcode.net/studying-the-angular-injector/))

In order for the Injector to know what to inject into a given functions parameters, it needs a list of these parameters. This is what the `annotate` function does.

There are three different ways in Angular to annotate your methods.

- Use an array. The last element of the array is the function, the rest is a list of the parameter names. 
```language-javascript
['$scope', '$q', function($scope, $q) {}
``` 

- Add a `$inject` property to the function containing a list of the parameters. 
```language-javascript
function myFunction($scope, $q) {}
myFunction.$inject = ['$scope', '$q'];
```
- Nothing at all. Angular can parse the parameters straight out of the function. This does not work when your code is minified since the minification renames your parameters to the smallest length possible.
```language-javascript
function myFunction($scope, $q) {}
```

Lets look at the code.

```language-javascript
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG_SPLIT = /,/;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg;
var $injectorMinErr = minErr('$injector');
function annotate(fn) {
  var $inject,
      fnText,
      argDecl,
      last;

  if (typeof fn == 'function') {
    if (!($inject = fn.$inject)) {
      $inject = [];
      if (fn.length) {
        fnText = fn.toString().replace(STRIP_COMMENTS, '');
        argDecl = fnText.match(FN_ARGS);
        forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg){
          arg.replace(FN_ARG, function(all, underscore, name){
            $inject.push(name);
          });
        });
      }
      fn.$inject = $inject;
    }
  } else if (isArray(fn)) {
    last = fn.length - 1;
    assertArgFn(fn[last], 'fn');
    $inject = fn.slice(0, last);
  } else {
    assertArgFn(fn, 'fn', true);
  }
  return $inject;
}
```


`annotate` is passed either an array or a function.

If an array has been passed into `annotate`, it is assumed to be in the first form (`['$scope', function($scope) {..`). In this case, we assert that the last element in the array is a function and then remove that element returning the array.

If it is just a function passed in, first it checks if the function has already been annotated by setting a `$inject` property.
```language-javascript
    if (!($inject = fn.$inject)) {
```

If it hasn't been, we need to extract the parameters from the function definition itself. This part of the code gives us an enjoyable journey through the power of regular expressions.
```language-javascript
      $inject = [];
      if (fn.length) {
        fnText = fn.toString().replace(STRIP_COMMENTS, '');
        argDecl = fnText.match(FN_ARGS);
        forEach(argDecl[1].split(FN_ARG_SPLIT), function(arg){
          arg.replace(FN_ARG, function(all, underscore, name){
            $inject.push(name);
          });
        });
      }
```

Angular takes advantage of the fact that calling `toString()` on a function definition returns the full text of the function. We can see this in the nodejs repl:
```language-javascript
> var x = function(a,b,c) { };
undefined
> x.toString();
'function(a,b,c) { }'
```

`toString()` returns the function exactly as it has been entered in the script, including comments.

These comments get in the way of parsing the parameters, so first the method strips any comments from the function. Anything that matches the regex `/((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg` is removed. The first part of that regex (`(\/\/.*$)`) matches anything starting with // up to the end of the line. The second part (`(\/\*[\s\S]*?\*\/)`) matches anything between a /* and */. 

Let's see if this works :

```language-javascript
>var fn = function(c/*a parameter*/, a, // More parameters
...                 /* Interesting */ r) { };
> fn.toString().replace(/((\/\/.*$)|(\/\*[\s\S]*?\*\/))/mg, '');
'function (c, a,
                  r) { }'
```

It works, the comments are stripped out.

Next it pulls out the argument list with the regex match : `/^function\s*[^\(]*\(\s*([^\)]*)\)/m`

This regex is bordering on the complex. Let's break it down.

 * **^** : From the start of the string
 * **function** : match the text *function*
 * **\s*** : any amount of whitespace
 * **[^\\(]*** : any character that is not a '('. This will be whitespace and the function name.
 * **\\(** : the '(' character. Note ( is a special char in regular expressions, so the '\' is needed to escape it and make the regex treat it as a real character.
 * **\s*** : more whitespace
 * **([^\)]*)** : Any character that is not a ')'. Also note that this part of the pattern is within '(' and ')'. This is a group and means we are especially interested in what is matched here.
 * **\)** : The closing ')'.
 
Does it work?

```language-javascript
>fnText.match(/^function\s*[^\(]*\(\s*([^\)]*)\)/m)
[ 'function (c, a, \r\n                  r)',
  'c, a, \r\n                  r',
  index: 0,
  input: 'function (c, a, \r\n                  r) { }' ]
```

`match` returns an array of the matches found by the regex. The first element in the array is the whole text matched. Each ensuing element is each group found within the regex. The group matched here is anything between the brackets after the text 'function'. In this case it is the text `'c, a, \r\n                  r',` which is our parameter list (plus the newlines and white space).

The code then splits this string at the commas. For each of these then does a further call to replace.
```language-javascript
arg.replace(FN_ARG, function(all, underscore, name){
            $inject.push(name);
          });
```

Now looking at the documentation for [replace](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace) you are supposed to call this function with the search string and a function which is supposed to return the string that you replace it with. This code doesn't do this. Instead it just takes the matched string and pushes it into the `$inject` array, ignoring any return value. An interesting and neat way to find matches within text.

Lets look at the regex used to match these parameters `/^\s*(_?)(\S+?)\1\s*$/` :

 * **^** : the beginning of the parameter text
 * **\\s*** : any amount of whitespace
 * **(__?)** : 0 or more '__' characters.
 * **(\S+?)** : 1 or more non-whitespace characters. The ? makes it a lazy match, it will match as few characters as possible - so this will stop matching when we get to the underscores defined in the next section.
 * **\1** : This matchs the substring that is the same as that in group 1. This is the underscores. The effect of this is to ensure that if our parameter starts with a certain number of _ characters that it will end with the same amount.
 * **\s*** : any amount of whitespace

An interesting point to take away from this pattern is that in your Angular module you can surround your parameters with any number of underscores and (as long as there is the same amount on either side) these will not be included in the annotation and consequent service location.

Phew, that's a lot of regex.

We should now have a list of the parameters which the injector can then use to locate the values to inject into our function.

Next: [invoke](http://taoofcode.net/studying-the-angular-injector-invoke)