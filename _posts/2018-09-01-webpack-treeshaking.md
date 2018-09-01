---
title: 'Modernizing a webpack/babel config w/ esm treeshaking'
category: programming
tags: [webpack]
---

Hearing so much about treeshaking in webpack, (and rollup, and parcel), it would
seem reasonable that it'd be a well documented type of migration.

[The webpack docs][wp-treeshaking] have a section on tree shaking, where it looks
alluringly simple. We just have to use `import { thing } from 'lib';` everywhere,
and reap the blessings of the webpack gods.

If, however, we are using plugins like [babel][babel] in our webpack configuration,
(and honestly, who doesn't?) there are several things that can prevent the tree
shaking from working.

**TLDR:** After a couple days of trying, failing and trying again, the traps seem to be:

- We now want to transpile everything. To process those sweet new esm bundles.
- Don't transpile module types, i.e. make sure babel doesn't change esm to commonjs.
- Don't [inject `import`s via transforms][transform-runtime-issue], e.g. `polyfill`s,
  since it messes with module type detection.
- Don't mix esm `import` and commonjs `module.exports`, they are incompatible.


# Transpile everything!

Wait, what? The last time I checked, it was common practice to not transpile
`node_modules` w/ babel, for both performance and compatibility reasons. The assumption
being that libraries ship in a precompiled state, for their target. Whether that
is `nodejs`, `latest chrome`, or something like electron. Granted, that assumption might
have been naive, and it is part of what's keeping the community from moving forward.

Our first `webpack.config.js` rule:

```js
[
	{
		test: /\.js$/,
		exclude: /node_modules/,
		use: ['babel-loader']
	}
];
```

We have a problem: if we want the esm version of our dependencies,
they cannot possibly come ready made for our (most likely) feature-lacking browsers.
By observing projects like `create-react-app` and `parcel`, we can get a pretty good
idea of how people go about handling this.

`parcel` uses a lot of magic to transpile `node_modules` based on the presence of
`.babelrc` files in the individual packages, while merging it with a default
transform: `preset-env`. This alleviates the need of setting up transforms ourselves.

The common tendency here is to have a babel config for your app, and another with
minimal transforms for `node_modules`.

Our separated `webpack.config.js` rule:

```js
[{
  oneOf: [
    {
      test: /\.js$/,
      exclude: /node_modules/,
      use: [
        // Config for our app code
        {
          loader: 'babel-loader',
          options: {
            // ...
          }
        }
      ]
    },
    {
      test: /\.js$/,
      use: [
        // Minimal transforms for 3rd party code
        {
          loader: 'babel-loader',
          options: {
            babelrc: false
            // ...
          }
        }
      ]
    }
  ];
}];
```

# Don't transpile module types. Watch those babel plugins!

As [the docs][wp-treeshaking] say, tree shaking relies on static `imports` and `exports`.
This means that we cannot get any help if what we're passing to webpack has already become
`commonjs` format.

What we want is this:

```js
import { map } from 'lodash';
map([1, 2, 3], x => x + 3);
```

But, if we're not careful with `babel`, what we're passing to webpack
works more like this (If curious, try it out on the [babel repl][babel-repl-lodash]):

```js
var map = require('lodash').map;
map([1, 2, 3], x => x + 3);
```

Which is not statically analyzeable by webpack, and we will not be able to get
it to remove the unnecessary code no matter how nicely we ask.

But what are we looking out for? Anything related to module formats.

## Let's inspect

For our purposes, we're migrating an existing webpack config, and so it is always
nice to start with seeing what's going on. What are we actually passing to webpack?
We can get an idea by adding [inspect-loader][inspect-loader] above (meaning after) `babel-loader` in
our config:

```js
[
	{
		test: /\.js$/,
		use: [
			// Config for our app code
			{
				loader: 'inspect-loader',
				options: {
					callback({ arguments: [sourceCode, { sources }] }) {
						console.log('// ', sources.join(', '), 'generated:');
						console.log(sourceCode);
					}
				}
			},
			{
				loader: 'babel-loader',
				options: {
					// ...
				}
			}
		]
	}
];
```

And then run our build:

```bash
$ webpack
```

```js
// src/index.js generated:
'use strict';

var _lodash = require('lodash');

(0, _lodash.map)([1, 2, 3], function(x) {
	return x + 3;
});
```

Which is no good, it's not the format webpack needs to do it's magic. Which magic flag
should we flip to get this to work?

## Presets and configs; @babel/preset-env

[@babel/preset-env][babel-preset-env] is probably the most pervasive preset in modern web development, which allows us to specify the browsers we want to support, rather
than which features we want to transpile. This offloads the task of figuring out
which flags to flip to support a given browser. Assuming we keep the packages up
to date.

If we set the modules option of preset-env to `false` in our `.babelrc`, or inline
in the webpack config, we can prevent the preset from handling modules for us:

```json
{
	"presets": [
		[
			"@babel/preset-env",
			{
				"modules": false
			}
		]
	]
}
```

If we try again by running `webpack`, our `inspect-loader` will now print:

```js
//  src/index.js generated:
import { map } from 'lodash';
map([1, 2, 3], function(x) {
	return x + 3;
});
```

which looks more promising.

> **Warning:** we need to check
> that none of the presets we add will try to do the module transpilation.
> For instance, as stated on the [preset-env][babel-preset-env-how] docs, the
> `babel-preset-stage-X` presets are **incompatible with `preset-env`**. Among the
> incompatible features are `esm` to `commonjs` modules transform. So removing it
> and similar plugins is essential to getting this to work.

# `javascript/esm` mode

We're not done yet though, since `esm` mode will create some new edge cases for modules.

If you encounter something like this:

> \> Uncaught TypeError: Cannot assign to read only property 'exports' of object '#<Object\>'

it's because `webpack` is determining the module format to be `esm`, but the module is trying to
use commonjs exports.

**If a file contains the `import` keyword** when they reach webpack,
it will assume it's an esm module, and run in `javascript/esm` mode, where `module.exports` is
`undefined`, and `exports` is read-only. So anywhere that uses `import` must use `esm` style
`export` as well. For more info see this [issue on webpack github][webpack-esm-issue]

This is potentially the biggest blocker to migrating our current codebase, since
using named exports everywhere (i.e. breaking `require('./foo')` by exporting
`.default`) will require major refactoring. Although that might be a task for a
nifty [codemod][jscodeshift], and another day.

[wp-treeshaking]: https://webpack.js.org/guides/tree-shaking/ 'Webpack tree shaking'
[webpack-esm-issue]: https://github.com/webpack/webpack/issues/4039#issuecomment-273804003
[inspect-loader]: https://github.com/peerigon/inspect-loader
[babel]: https://babeljs.io/ 'BabelJS'
[babel-preset-env]: https://www.npmjs.com/package/@babel/preset-env '@babel/preset-env on npm'
[babel-preset-env-how]: https://babeljs.io/docs/en/babel-preset-env#how-does-it-work 'How it works'
[babel-repl-lodash]: https://babeljs.io/en/repl.html#?babili=false&browsers=&build=&builtIns=false&spec=true&loose=true&code_lz=JYWwDg9gTgLgBAbziAhmOBfOAzKERwDkANhACYoDOAFoQNwBQqYAFANoCMANAExcDMAXS5wAHnAC8APjFwA1HH4BKOkA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=true&fileSize=false&timeTravel=false&sourceType=module&lineWrap=false&presets=latest%2Creact%2Cstage-2&prettier=false&targets=&version=6.26.0&envVersion= 'Lodash import on babel repl'
[transform-runtime-issue]: https://github.com/webpack/webpack/issues/4039#issuecomment-274094298
[jscodeshift]: https://github.com/facebook/jscodeshift
