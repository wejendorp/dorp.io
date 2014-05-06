---
layout: post
title:  "Global stylus variables with grunt-stylus"
date:   2014-05-05 22:47:03
categories: blog
---

[Stylus](http://learnboost.github.io/stylus/) is a CSS preprocessor that enables
the definition of various helpers in the form of mixins and variables for
configuration.

You should already know the awesome [nib](http://visionmedia.github.io/nib/)
stylus extensions. You might even have some helpers yourself, that you want to
be accessible in all your stylus files. Perhaps even included by default, saving
your the trouble of writing

```css
@import 'nib'
@import '../my/helpers'
```

In all your files.
Dont worry, `grunt-stylus` has you covered.

<!--more-->

## Path madness
Your stylus `@import` statements should be path independent.

For this purpose, we have the paths option. Think of it as your env `PATH`,
this is where stylus will look for your imports. The current directory is always
searched first, so don't worry it won't break your local references.

This means we can have unrelated (directory wise) imports at hand at all times,
and can reliably import the
[color variables]({% post_url 2014-05-03-shared-configurations-with-grunt-template %})
with `@import "dorp/colors"`, by configuring grunt-stylus target with:

```js
stylus: {
  all: {
    options: {
      paths: ['src/styles']
    }
    // files: { ... }
  }
}
```

## Automatic imports
Defining variables makes it easy to avoid dangling styles when changing the
general styles of your site. You can even save yourself the repetition of adding
the `@import`s to all your files by having them included automatically.

```js
stylus: {
  all: {
    options: {
      import: ['nib', ... ]
    }
    // files: { ... }
  }
}
```

*Gotcha*: Be aware that your chosen import names will first be included automatically
relative to your current file. So if your current directory contains a file by
the name `nib.styl`, this will be imported in place of the `nib` module, and
equivalently with any other imports you place here.

## Index files

Stylus works like your average web server, looking for `index.styl` in any folder
your try and import. This means you can have all your styles loaded by doing
`@import 'myStyles'`, if your `index.styl` file imports all its siblings.

<pre>
./myStyles
  |-- index.styl
  |-- vendor.styl
  |-- buttons.styl
  |-- images.styl
</pre>

Combine this with the import option array, and keep your styles organized and
automatically loaded.

*Gotcha*: Be aware not to have any style definitions in these files, only creating
vars and mixins. Otherwise your css output will contain these entries once for
each file you passed in.



## Conclusion

You can save yourself from a lot of headaches and repeated imports by defining
a global stylus path, and importing your helpers into all your files.
Even if you don't go the fully automatic way, make sure you define the lookup
paths to your styles dir, so you can forget all about relative paths.

Full example config, including `src/styles/dorp/index.styl` and `nib` automatically:

```js
grunt.initConfig({
  stylus: {
    compile: {
      options: {
        import: ['nib', 'dorp'],
        paths:  ['src/styles']
      },
      files: {
        'build/compiled_styles.css': [
          'src/directives/*/*.styl',
          'src/views/*/*.styl',
          'src/modules/**/*.styl'
        ]
      }
    }
  }
});
```
