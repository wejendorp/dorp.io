---
title:  "Extending $q promises in Angular"
tags:   [angular, promises]
redirect_from:
  - /2014/05/extending-q-promises/
  - /blog/extending-q-promises/
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
  uses `promise.then`.
* [deferred.reject](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L226-L228)
  uses `deferred.resolve`
* [deferred.notify](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L231-L245)
  does not use promises.

### promise
* [promise.catch](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L287-L289)
  uses `promise.then`.
* [promise.finally](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L291-L326)
  uses `deferred.reject` and `deferred.resolve`.

### `$q` methods:
  `$q.defer` is our normal point of entry.
* [$q.reject](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L384-L388)
  uses the internal `defer`
* [$q.when](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L421-L467)
  uses the internal `defer`.
* [$q.all](https://github.com/angular/angular.js/blob/35e53ca649c60a27272cac38e4e9d686fb0c74f2/src/ng/q.js#L495-L517)
  uses the internal `defer`.

It all comes down to the functions that produce a new "internal" promise via `defer`.


By hooking into `then`, `reject`, `when` and `all`,
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
      promise.then = function(thenFn, errFn, notifyFn) {
        var p = promise._then(thenFn, errFn, notifyFn);
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

    var defer = $delegate.defer,
        when = $delegate.when,
        reject = $delegate.reject,
        all = $delegate.all;
    $delegate.defer = function() {
      var deferred = defer();
      decoratePromise(deferred.promise);
      return deferred;
    };
    $delegate.when = function() {
      var p = when.apply(this, arguments);
      return decoratePromise(p);
    };
    $delegate.reject = function() {
      var p = reject.apply(this, arguments);
      return decoratePromise(p);
    };
    $delegate.all = function() {
      var p = all.apply(this, arguments);
      return decoratePromise(p);
    };

    return $delegate;
  });
});
```

With this configuration block in place, all promises in your application will
be able to listen in on the promise value without modifying it, using
`promise.success` and `promise.error`, without breaking when you extend the
chain with another `then`.
