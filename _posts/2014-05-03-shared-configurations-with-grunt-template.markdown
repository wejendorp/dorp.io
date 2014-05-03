---
layout: post
title:  "Shared configurations with grunt-template"
date:   2014-05-03 16:48:30
categories: blog
---

At [Mojn](http://mojn.com) we needed to use part of our shared environment
on the clientside app that uses [AngularJS](http://angularjs.org).

I stumbled upon [grunt-template](http://git.io/grunt-template) while browsing through npmjs looking
for interesting tasks. Grunt-template is a simple task designed to pass any data you provide into an
[underscore template](http://underscorejs.org/#template) file, and write the result to another file.

For our purposes we needed a way to write a reduced version of our server-side environment as an
angular module, as well as generate a shared color configuration in our stylus and script files.

# Angular modules
What if we could just define angular modules as <code>{myModule: {myValue: 'Some value'}}</code>?
This can be done with a simple template (<code>templates/angular-module.tpl</code>):

{% highlight javascript %}
<% _.forEach(modules, function(moduleData, moduleName) { %>
  angular.module('<%= moduleName %>',[])
  <% _.forEach(moduleData, function(v, k) { %>
    .constant('<%= k %>', <%= JSON.stringify(v) %>)
  <% }); %>;
<% }); %>
{% endhighlight %}

And the following grunt-template options:

{% highlight javascript %}
grunt.initConfig({
  template: {
    // define a new target called angular, to be run with grunt template:angular
    angular: {
      options: {
        data: {
          // Define your modules here
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
{% endhighlight %}

So now <code>grunt template:angular</code> should produce <code>dist/modules.js</code>:
{% highlight javascript %}
angular.module('myEnvironment')
  .const('env', {"name":"Development","basePath":"http://localhost:3000"})
  ;
{% endhighlight %}

<p></p>

# Stylus template
We want to define a <code>colors</code> module, for which we can use the angular template,
and a stylus file that defines the vars.

{% highlight javascript %}
<% _.forEach(stylus.variables, function(colorCode, color) { %>
<%- color %> = <%- colorCode %>;<% }); %>
{% endhighlight %}

Using the same setup as the previous

{% highlight javascript %}
// import or define your colors
var colors = {
  blue: '#00F',
  red: '#F00'
};

grunt.initConfig({
  template: {
    // define a new target called colors, to be run with grunt template:angular
    colors: {
      options: {
        data: {
          modules: {
            colors: {
              colors: colors
            }
          },
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
{% endhighlight %}

<code>grunt template:colors</code> will then produce

<code>dist/colors.js</code>
{% highlight javascript %}
angular.module('colors')
  .const('colors', {"blue": "#00F", "red": "#F00"})
  ;
{% endhighlight %}

<code>dist/colors.styl</code>
{% highlight sass %}
red = #00F;
blue = #F00;
{% endhighlight %}
