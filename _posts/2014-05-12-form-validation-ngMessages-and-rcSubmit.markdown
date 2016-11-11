---
layout: post
title:  "Form validation with rcSubmit and bootstrap"
date:   2014-05-12 23:42:43
tags: 	angular validation rcSubmit bootstrap
---

I have long been annoyed with doing form validations in Angular, it just never
felt right. I'd manually link every input to a form with some convoluted
`ng-class` conditions, not to mention track form state in controllers and
polluting the scopes with loading indicators.

I just recently found
[an excellent writeup](http://www.yearofmoo.com/2014/05/how-to-use-ngmessages-in-angularjs.html)
on using the new `ngMessages` module in Angular `>=v1.3.0.beta.8` have your
error messages prioritized and easily customized. This made me rethink and
rewrite the way I use forms.

Which brought me back to [rcSubmit](http://code.realcrowd.com/on-the-bleeding-edge-advanced-angularjs-form-validation/)
and [the source](https://github.com/realcrowd/angularjs-utilities/blob/master/src/directives/rcSubmit.js).
With `rcSubmit` the state tracking and loading indicators are handled for us,
but spamming my forms with

```
ng-class="{has-error: rc.form.needsAttention(form.field)}"
```

feels too repetitive.

If we make a few assumptions on form structure this can easily be avoided with
a few directives.



## Too much template logic, too repetitive
With even simple validations, there is much too much logic written in markup.
Notice how many references we're making to the form in order to validate a
single field.

```html
<form rc-submit="myAction()" name="myForm">
  <div class="form-group"
       ng-class="{has-error: rc.myForm.needsAttention(myForm.myField)}">

    <label for="emailAddress">Enter your email address:</label>
    <input type="email"
           name="emailAddress"
           ng-model="data.email"
           ng-minlength="5"
           ng-maxlength="30"
           required />

    <div ng-messages="">...</div>
  </div>
  <button class="btn btn-default"
          type="submit"
          ng-disabled="rc.myForm.submitInProgress">Reset password</button>
</form>
```

Since we're making this for bootstrap forms, we'll make some assumptions on the
structure of our forms:

```html
<form rc-submit="myAction()" name="myForm">
  <div class="form-group">
    <input class="form-control" ng-model="myModel" />
    <div ng-messages="">...</div>
  </div>
  <button class="btn" type="submit">Submit</button>
</form>
```

Which will let us create a chain of directives.

## Disabling submit button on load

Let's start simple, and get rid of the `ng-disabled` attribute on our submit
button. We'll probably want this on all our forms, so creating a directive with
a semantic name will make the template cleaner to read.

By require'ing `^rcSubmit` we'll get access to `rcSubmitController` of the
parent form.

```js
angular.module('myApp')
.directive('formSubmit', function() {
  return {
    restrict: 'C',
    require: '^rcSubmit',
    link: function(scope, element, attrs, controller) {
      scope.$watch(function() {
        return controller.submitInProgress;
      }, function(val) {
        element.toggleClass('disabled', val);
        element.attr('disabled', !!val);
      });
    }
  };
});
```
This will make any `button.form-submit` disabled when the form it belongs to is
submitting. Note that this only works if your submit action returns a promise
for `rcSubmit` to wait for.

With this directive in place our submit button no longer references the form in
the template. One reference down, two to go.

```html
<form rc-submit="myAction()" name="myForm">
  ...
  <button class="btn btn-default form-submit"
        type="submit">Reset password</button>
</form>
```

## Setting error class on the form group
Let's get rid of the `ng-class` on the form group, and two more form references.

```html
<div class="form-group"
     ng-class="{has-error: rc.myForm.needsAttention(myForm.myField)}">
  <input class="form-control" ng-model="myModel" />
</div>
```

We can use the `form[rc-submit] > .form-group` relation in the same way we just
did for the form submit button.

Unlike our formSubmit directive, we'll want to make this directive fail
silently, since `.form-group` might have nothing to do with an `[rc-submit]`
form. So we'll make the controller optional in `require`.

We want to create a directive controller that allows the inputs to register
themselves as part of a validation (form-)group.

```js
angular.module('myApp')
.directive('formGroup', function() {
  return {
    restrict: 'C',
    require: ['^?rcSubmit', 'formGroup'],
    controller: function() {
      this.inputs = [];
      this.addInput = function(input) {
        this.inputs.push(input);
      };
    },
    link: function(scope, element, attrs, controllers) {
      // ...
    }
  };
});
```

An input can refenrence the `formGroup` controller and call `addInput` to
register. Let's hook up the bootstrap `.form-control` to do this.
Again, keeping the dependency soft, so we won't break any forms that don't use
rcSubmit.

We'll just load up the `ngModel` and register it with `formGroup` if possible.

```js
angular.module('myApp')
.directive('formControl', function() {
  return {
    restrict: 'C',
    require: ['ngModel', '^?formGroup'],
    link: function (scope, element, attr, controllers) {
      var modelCtrl = controllers[0];
      var groupCtrl = controllers[1];

      if(groupCtrl)
        groupCtrl.addInput(modelCtrl);
    }
  };
});
```

So now our `formGroup` directive should contain a list of all it's contained
`formControl` elements.

Now we need to validate them all. Recall the `rcSubmit` needsAttention method,
let's just apply that to each model and set the error if any of them fail.

```js
  // ...
  link: function(scope, element, attrs, controllers) {
    var formCtrl  = controllers[0];
    var groupCtrl = controllers[1];

    // option to disable via novalidate attribute
    // and only run if the rcSubmit controller was found
    if(typeof attrs.novalidate !== 'undefined' || !formCtrl) return;

    var fn = function() {
      return groupCtrl.inputs.reduce(function(accum, field) {
        return accum || formCtrl.needsAttention(field);
      }, false);
    };

    scope.$watch(fn, function(val) {
      if(typeof val === 'undefined') return;
      element.toggleClass('has-error', val);
    });
  }
```

Now we will have the error indication with just the markup:

```html
<form rc-submit="myAction()" name="myForm">
  <div class="form-group">
    <input class="form-control" ng-model="myModel" />
  </div>
</form>
```

Which has made our markup entirely free of form references.


```html
<form rc-submit="myAction()" name="myForm">
  <div class="form-group">

    <label for="emailAddress">Enter your email address:</label>
    <input type="email"
           name="emailAddress"
           ng-model="data.email"
           ng-minlength="5"
           ng-maxlength="30"
           required />
    <div ng-messages="">...</div>
  </div>
  <button class="btn btn-default form-submit"
          type="submit">Reset password</button>
</form>
```

## What about ng-messages?
We can get the list of fields belonging to a form group by requireing the
`formGroup` controller. And then loop the equivalent `formField.$error`
objects in an ng-repeat.

If your group contains multiple fields and you want to customize the message
for each of them, you should be able to override this discovery and just tell
it which field to use. For this we'll use the `for` attribute.

If we attempt to do
```js
scope.fields = [formCtrl[attr.for]];
```

we'll run into issues if we try to define a message before it's field in the
form. We can solve this by using a watcher.

```js
angular.module('myApp')
.run(function($templateCache) {
  $templateCache.put('validations.default',
    '<div ng-message="required">You left the field blank...</div>'+
    '<div ng-message="minlength">Your field is too short</div>'+
    '<div ng-message="maxlength">Your field is too long</div>'+
    '<div ng-message="email">Your field has an invalid email address</div>'
  );
})
.directive('validationMessages', function() {
  return {
    restrict: 'EA',
    require: ['^form', '^formGroup'],
    scope: {},
    priority: 100,
    link: function(scope, element, attr, controllers) {
      var formCtrl  = controllers[0];
      var formGroup = controllers[1];

      scope.fields = formGroup.inputs
                      .map(function(i) {
                        return formCtrl[i.$name];
                      });

      if(attr.for)
        scope.$watch(
          function() { return formCtrl[attr.for]; },
          function(field) { scope.fields = [field]; }
        );
    },
    transclude: true,
    template:
      '<div ng-repeat="field in fields" ng-messages="field.$error" '+
      'ng-messages-include="validations.default" ng-transclude></div>'
  };
});

```

Your form is now completely free of name ties, and you've put your directives
to work for you.

```html
<form rc-submit="myAction()" name="myForm">
  <div class="form-group">

    <label for="emailAddress">Enter your email address:</label>
    <input type="email"
           name="emailAddress"
           ng-model="data.email"
           ng-minlength="5"
           ng-maxlength="30"
           required />

    <!-- equivalent -->
    <div validation-messages for="email"></div>
    <div validation-messages></div>
  </div>
  <button class="btn btn-default form-submit"
          type="submit">Reset password</button>
</form>
```



## Source

```js
angular.module('myApp')
.directive('formGroup', function() {
  return {
    restrict: 'C',
    require: ['^?rcSubmit', 'formGroup'],
    controller: function() {
      this.inputs = [];
      this.addInput = function(input) {
        this.inputs.push(input);
      };
    },
    link: function(scope, element, attrs, controllers) {
      var formCtrl  = controllers[0];
      var groupCtrl = controllers[1];

      if(typeof attrs.novalidate !== 'undefined' || !formCtrl) return;

      var fn = function() {
        return _.reduce(groupCtrl.inputs, function(accum, field) {
          return accum || formCtrl.needsAttention(field);
        }, false);
      };

      scope.$watch(fn, function(val) {
        if(typeof val === 'undefined') return;
        element.toggleClass('has-error', val);
      });
    }
  };
})
.directive('formControl', function() {
  return {
    restrict: 'C',
    require: ['ngModel', '^?formGroup'],
    link: function (scope, element, attr, controllers) {
      var modelCtrl = controllers[0];
      var groupCtrl = controllers[1];

      if(groupCtrl)
        groupCtrl.addInput(modelCtrl);
    }
  };
})
.directive('formSubmit', function() {
  return {
    restrict: 'C',
    require: '^rcSubmit',
    link: function(scope, element, attrs, controller) {
      scope.$watch(function() {
        return controller.submitInProgress;
      }, function(val) {
        element.toggleClass('disabled', val);
        element.attr('disabled', !!val);
      });
    }
  };
})
.run(function($templateCache) {
  $templateCache.put('validations.default',
    '<div ng-message="required">You left the field blank...</div>'+
    '<div ng-message="minlength">Your field is too short</div>'+
    '<div ng-message="maxlength">Your field is too long</div>'+
    '<div ng-message="email">Your field has an invalid email address</div>'
  );
})
.directive('validationMessages', function() {
  return {
    restrict: 'EA',
    require: ['^form', '^formGroup'],
    scope: {},
    priority: 100,
    link: function(scope, element, attr, controllers) {
      var formCtrl  = controllers[0];
      var formGroup = controllers[1];

      scope.fields = formGroup.inputs
                      .map(function(i) {
                        return formCtrl[i.$name];
                      });

      if(attr.for)
        scope.$watch(
          function() { return formCtrl[attr.for]; },
          function(field) { scope.fields = [field]; }
        );
    },
    transclude: true,
    template:
      '<div ng-repeat="field in fields" ng-messages="field.$error" '+
      'ng-messages-include="validations.default" ng-transclude></div>'
  };
});
```
