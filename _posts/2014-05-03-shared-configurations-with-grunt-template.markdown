---
title: "Shared configurations with grunt-template"
tags: [grunt]
redirect_from:
  - /2014/05/shared-configurations-with-grunt-template/
  - /blog/shared-configurations-with-grunt-template/
---

At [Mojn](http://mojn.com) we needed a way to write a reduced version of our server-side environment as an
angular module, as well as generate a shared color configuration in our stylus and script files.

I stumbled upon [grunt-template](http://git.io/grunt-template) while browsing through npmjs looking
for interesting tasks. Grunt-template is a simple task designed to pass any data you provide into an
[underscore template](http://underscorejs.org/#template) file, and write the result to another file.
That's perfect! Now let's write some templates and configure grunt-template.</section>



## Angular modules
What if we could just define angular modules as `{myModule: {myValue: 'Some value'}}`?
This can be done with this simple template.

```js
// templates/angular-module.tpl
<% _.forEach(modules, function(moduleData, moduleName) { %>
  angular.module('<%= moduleName %>',[])
  <% _.forEach(moduleData, function(v, k) { %>
    .constant('<%= k %>', <%= JSON.stringify(v) %>)
  <% }); %>;
<% }); %>
```

And the following grunt-template options:

```js
// Gruntfile.js
grunt.initConfig({
  template: {
    // define a new target called angular
    angular: {
      options: {
        data: {
          // Define modules here
          modules: {
            myEnvironment: {
              env: {
                name: 'Development',
                basePath: 'http://localhost:3000'
              }
            }
          }
        }
      },
      files: {
        'dist/modules.js': 'templates/angular-module.tpl'
      }
    }
  }
});
```

So now `grunt template:angular` should produce

```js
// dist/modules.js
angular.module('myEnvironment')
  .const('env', {"name":"Development","basePath":"http://localhost:3000"})
  ;
```

ready to be included in the angular app, and point to the environment path as `env.basePath`.


## Stylus template
We want to define a `colors` module, for which we can use the angular template,
and a stylus file that defines the vars. How about something like this:

```js
// templates/stylus.tpl
<% _.forEach(stylus.variables, function(value, key) { %>
<%- key %> = <%- value %>;<% }); %>
```

That should make a variable for every value in `stylus.variables`, passed to the template.
Now we just need to pass an object, `{stylus: {variables: { ... }}}` to the
task and point to the newly created template.

```json
// colors.json
{
  "blue": "#00F",
  "red":  "#F00"
}
```

```js
// Gruntfile.js
var colors = require('./colors.json');

grunt.initConfig({
  template: {
    // ...
    // define a new target called colors
    colors: {
      options: {
        data: {
          // For use with angular-module.tpl
          modules: {
            colors: {
              colors: colors
            }
          },
          // For use with stylus.tpl
          stylus: {
            variables: colors
          }
        }
      },
      files: {
        // reuse angular template from previous section
        'dist/colors.js'  : 'templates/angular-module.tpl',
        'dist/colors.styl': 'templates/stylus.tpl'
      }
    }
  }
});
```

`grunt template:colors` should now produce

```js
// dist/colors.js
angular.module('colors')
  .const('colors', {"blue": "#00F", "red": "#F00"})
  ;
```

```css
// dist/colors.styl
red = #00F;
blue = #F00;
```

and we're ready to use our color scheme in code as well as style files.
