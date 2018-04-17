---
title: "Angular from a React perspective"
category: programming
tags: [angular]
---

This post is written for the purpose of covering some gotchas and observations when going back to using
Angular 1.x. For reference, we'll use `react` components and lifecycles.

This might seem backwards, but has come up when returning to a large legacy codebase.


## Digest cycles and blessed `$services`
Angular, unlike React, uses a digest loop to figure out when things have changed. This means it's
possible to write small mutations to objects, and then wait for (or trigger) the checker, which will
update the views accordingly.

As long as we're doing updates in synchronous code, angular will automatically pick up our changes.
If we are using asynchronous code, such as doing XHR or `setTimeout`, you need to tell angular that
something has happened.

Angular provides a set of services that do the digest stuff for us such as `$http` and `$q`, but If
we are doing async updates outside of these, we need to trigger angular. If we do `setTimeout` we're
basically leaving angular land. In order to get back in, we can use

- `$scope.$digest` which will trigger a digest cycle.
- `$scope.$apply` which takes an expression to evaluate in the current scope, in order to trigger a
digest.

Be aware that triggering a digest cycle in the middle of an existing cycle will throw an error. It is commonly handled by implementing a `$safeDigest` or `$safeApply` method, which checks the `$phase` before doing so.

Handling events from outside the digest cycle will also come up if we're implementing a wrapper for
a third party library such as a jQuery based date picker, or anything using document events.
Unlike react, angular does not abstract and handle DOM events as synthetic events. The most common
ones are covered by directives such as `ngFocus`, but binding via DOM methods will cause the callback
to happen outside of angular land.

Take care not to create side-effect loops from the `$digest` cycle.

<!-- This feature can be emulated in React by implementing `componentShouldUpdate` functions (maybe?). -->


## `ngHTML`: It's all in the directives
In `react`, every component is either a pure function or a class, with it's associated markup inside.
Everything is being handled by the `react-dom` library, which transforms the components into DOM elements.

In angular, things get a bit more fuzzy. `ngHTML` works as a superset of HTML, with the angular directives
added in. This means when we're writing our templates, we can use things like `ng-if` to create a
subtree that will only render if a condition is true. But how does this actually work?

Whenever we encounter an `ng-` attribute, or custom element tag, it's probably a directive. Directives
are much more than "expanding" elements though.

###  Directive types: Elements, attributes and classes
Angular supports more kinds of directives than the `element` type known from react. This is expressed
in directives as `restrict: 'EAC'`.

- Elements, or `E` will make the directive work as `<MyDirective></MyDirective>`, and is
- Attributes, or `A` will make the directive work as `<div my-directive></div>`.
- Classes, or `C` will make the directive work as `<div class="my-directive"></div>`.

### Directive scopes: Passing around values
Where in react we're passing down props, in angular everything is handled via `$scope`.
A directive can have inherited scope, isolated scope or different bindings.
Scopes defined with the binding keys is how we pass values to our directives.

Some directives create new scopes, which might create hard to debug situations.
This comes up with [`ngIf`](https://docs.angularjs.org/api/ng/directive/ngIf) for instance, which
creates a new scope for


```js
angular.module('docsTabsExample', [])
.directive('myTabs', function() {
  return {
    restrict: 'E',
    scope: {},
  };
});
```



### Triggering actions on change
In react land, we have `componentWillReceiveProps`, `componentWillUpdate` etc.
`$scope.$watch` serves this purpose.


One thing to be aware of when writing mutating code is the depth of the `$watch`. We can tell angular
to `$watch` deep on the list of objects that we're rendering, and while that will work for most cases,
it's also extremely slow.

In the same way that we try to move logic and state up the tree in react, we should try to move state
and logic into services as much as possible.


## Scope inheritance gotcha w/ two-way bindings
When writing a controller that binds to `$scope.username` or similar, and wondered why
the value would not properly propagate back to the parent, this is for you.

A common use case for this is `ng-model`, which does a two-way binding by default.

Can best be explained by noting that scopes inherit values via [prototypical inheritance](https://github.com/angular/angular.js/wiki/Understanding-Scopes#javascript-prototypal-inheritance).

```js
// Angular has `$rootScope` instance, which we'll define here:
const $rootScope = {};

// Let's define the username globally in our app:
$rootScope.username = 'test-user';

const $scope = Object.create($rootScope);
// Each scope gets a handle to it's parent via `$scope.__proto__`:
// $scope.__proto__ = $rootScope

// When we use this in a two-way binding, we set the property in the wrong place:
$scope.username = 'hi';

console.assert($scope.user.username === $rootScope.user.username, 'expected usernames to match');
// Assertion failed: expected usernames to match
```

What happened? Instead of going up the prototype chain and finding the existing property, javascript
will set the property on the child object. It was simply being lazy. We can force a lookup up the chain,
either by looking up a non-primitive value, or via reassignment (e.g. `$scope.a += 'b';`).
<!-- TODO: YDKJS ref. -->

```js
// Let's define the username globally in our app, on an object instead of a primitive:
$rootScope.user = {};
$rootScope.user.username = 'test-user';

const $scope = Object.create($rootScope);

// Now when we set the property;
$scope.user.username = 'hi';
console.assert($scope.user.username === $rootScope.user.username, 'expected usernames to match');
// It works!
```

This is why we want to be passing objects, not primitives around to other scopes and directives whenever
possible. By doing so, we avoid creating weird disconnects if we're doing mutations.

Of course, nothing prevents us from writing angular directives and controllers using the unidirectional
dataflow principles from react, which makes these issues irrelevant.

## Dependency injection and abstractions
Angular comes with a whole set of abstractions for creating and configuring services, values and
templates.

### Dependency injection annotation
Angular uses a bit of magic to figure out what our controllers and services need. It will try to read
the function signature

```js
module.exports = function myController($scope) {};
```
And inject the `$scope` instance. However, when doing **minification** of javascript, this might become

```js
module.exports = function fn(a) {};
```
Which will trigger a message like `Unknown service 'a'`.

In order to write code that will survice minification, we have two equivalent options:
```js
//
module.exports = ['$scope', function myController($scope) {}];

// Or using $inject if you prefer to set a property to passing around arrays
function myController($scope) {};
myController.$inject = ['$scope'];
module.exports = myController;
```

How does this survive minification when the variable names do not? Simple, `uglifyjs` etc will not
touch string values and property names.

### Services, factories, what? Different service abstractions

A `service` let's us write the service definition as a constructor function. This means methods are
are added via `this` context.

```js
app.service('ping', ['$http', function($http) {
  this.request = function request() {
    return $http.send('/ping');
  };
}]);
```

A `factory` let's us write a service as a factory method, which will return the service in the end.

```js
app.factory('ping', ['$http', function($http) {
  // Note the return
  return {
    request() {
      return $http.send('/ping');
    }
  };
}]);
```

A `provider` let's us go up another level, and write a function that'll return the factory. This is
used to allow configuration of the service before it is created. For instance by setting the `$routeProvider`
to use html5 `pushState` instead of `hash` routing, changing the bindings it'll do when created.

Here we can make the ping url configurable in the `config` phase, but hidden during the `run` phase
inside the function scope.

```js
function pingProvider() {
  let pingUrl = '/track';

  // A provider method for configuring where the ping events should be sent
  this.setPingUrl = function(url) {
    pingUrl = url;
  };

  // The service factory function
  this.$get = ['$http', function($http) {
    return {
      // Call this to save the tracked events to the pingUrl
      request() {
        $http.send(pingUrl);
      }
    };
  }];
}

app.provider('ping', pingProvider);
// Configure it early
app.config(($pingProvider) => {
  $pingProvider.setPingUrl('/hello');

// $get instance
}).run(($ping) => {
  $ping.request();
});
```

So we have three layers of abstraction at our disposal when creating a service. Consider carefully
what kind of control you need, and choose the least powerful/complex option that covers our usecase.

### Config phases
We can only access a `$provider` type in the `config` step of your application. Everything else is
available for injection during the `run` phases. To use the example above, this prevents us from
accessing `$pingProvider` in a controller.

This will come up when using certain angular libraries such as `ng-router`, or when doing advanced
configuration of services.
