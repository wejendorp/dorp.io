---
layout: post
title:  "Extending $q promises in Angular"
date:   2014-05-20 08:54:00
categories: blog
---

## Why another post about this?
Most implementations seem to overlook the fact that the changes applied to
`delegate.defer` only affect the first promise in the chain, since
by design the `defer` function used internally in Angular cannot be modified.

* [(SA) How can I extend $q promise in Angularjs with a .succes and .error](http://stackoverflow.com/a/17889426)
* [(GH) Decoration of $q defer.promise](https://github.com/angular/angular.js/issues/5998)

Since we can affect the first returned promise we have our way in. Now to make
sure we stay "in".

Lets break it down by going through [the source](https://github.com/angular/angular.js/blob/master/src/ng/q.js).

### deferred:
* [deferred.resolve](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L207-L223)
  uses `then`.
* [deferred.reject](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L226-L228)
  uses `resolve`
* [deferred.notify](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L231-L245)
  does not use promises.

### promise
* [promise.catch](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L287-L289)
  uses `promise.then`.
* [promise.finally](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L291-L326)
  uses `deferred.reject` and `deferred.resolve`.

### `$q` methods:
* [$q.reject](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L384-L388)
  uses `deferred.reject`.
* [$q.when](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L421-L467)
  uses a mix of `deferred.resolve` and `promise.then`.
* [$q.all](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L495-L517)
  uses `deferred.resolve`, `deferred.reject` and `promise.then`.

So it seems everything falls back on `promise.then`.

By hooking into `then`,
and decorating every created promise before returning it, we should
never have to encounter another non-decorated promise.

## Example
For our purposes, lets add non-intercepting callbacks like the ones found on
[$http](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/http.js#L654-L666).

```js
angular.module('myApp').config(function($provide) {
  $provide.decorator('$q', function($delegate) {
    // Extend promises with non-returning handlers
    function decoratePromise(promise) {
      promise._then = promise.then;
      promise.then = function(thenFn, errFn) {
        var p = promise._then(thenFn, errFn);
        return decoratePromise(p);
      };

      promise.success = function (fn) {
        promise.then(function (value) {
          fn(value);
        });
        return promise;
      };
      promise.error = function (fn) {
        promise.then(null, function (value) {
          fn(value);
        });
        return promise;
      };
      return promise;
    }

    var defer = $delegate.defer;
    $delegate.defer = function() {
      var deferred = defer();
      decoratePromise(deferred.promise);
      return deferred;
    };

    return $delegate;
  });
});
```

With this configuration block in place, all promises in your application will
be able to listen in on the promise value without modifying it, using
`promise.success` and `promise.error`, without breaking when you extend the
chain with another `then`.
