---
title: "Modernizing Webpack and babel config w/ ESM treeshaking"
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
* We now want to transpile everything. To process those sweet new ESM bundles.
* Don't transpile modules! E.g. make sure babel doesn't change ESM to commonjs
* Don't inject `import`s, e.g. `babel-polyfill`s, since it messes with module type
detection.


# Transpile everything!
Wait, what? The last time I checked, it was common practice to not transpile
`node_modules` w/ babel, for both performance and compatibility reasons. The assumption
being that libraries ship in a precompiled state, for their target. Whether that
is `nodejs`, `latest chrome`, or something like electron. Granted, that assumption might
have been naive, and it is part of what's keeping the community from moving forward.

```js
{
  test: /\.js$/,
  exclude: /node_modules/,
  use: [
    'babel-loader'
  ]
}
```

We have a problem: if we want the ESM version of our dependencies,
they cannot possibly come ready made for our (most likely) feature-lacking browsers.
By observing projects like `create-react-app` and `parcel`, we can get a pretty good
idea of how people go about handling this.

The common tendency here is to have a babel config for your app, and another with
minimal transforms for `node_modules`.

```js
oneof: [{
  test: /\.js$/,
  exclude: /node_modules/,
  // Config for our app code
  use: [{
    loader: 'babel-loader',
    options: {
      // ...
    }
  }]
}, {
  test: /\.js$/,
  // Minimal transforms for 3rd party code
  use: [{
    loader: 'babel-loader',
    options: {
      babelrc: false
    }
  }]
}]
```

# Don't transpile module types. Watch those babel plugins!
As [the docs][wp-treeshaking] say, tree shaking relies on static `imports` and `exports`.
This means that we cannot get any help if what we're passing to webpack has already become
`commonjs` format.

What we want is this:

```js
import { map } from 'lodash';
map([1,2,3], x => x + 3);
```

But, if we're not careful with `babel`, what we're passing to webpack
works more like this (If curious, try it out on the [babel repl][babel-repl-lodash]):

```js
var map = require('lodash').map;
map([1,2,3], x => x + 3);
```

Which is not statically analyzeable by webpack, and we will not be able to get
it to remove the unnecessary code no matter how nicely we ask.

But what are we looking out for? Anything related to module formats.

## Let's inspect
For our purposes, we're migrating an older webpack config
webpack-inspect-loader

We can't really do the same w/ babel.. or can we?
Babel plugin ordering?


## [@babel/preset-env][babel-preset-env]
Probably the most pervasive preset in modern web development, which allows us to
specify the browsers we want to support, rather than which features we want to
transpile. This offloads the task of figuring out which

```json
{
  "presets": [
    ["env", {
      "targets": {
        "browsers": ["last 2 versions"]
      }
    }]
  ]
}
```


Caveats with tree shaking, webpack and babel...
getting that sweet ESM!

https://twitter.com/wejendorp/status/1035072104834719745

Disable all the `module` transpilations!?

First hint at issue:
https://babeljs.io/docs/en/babel-preset-env#how-does-it-work

THey probably have it figured out:
https://github.com/facebook/create-react-app/blob/next/packages/babel-preset-react-app/create.js


preset-env + plugins for things that are not stable features yet..
object-rest-spread, ...?

# Don't confuse the webpack. Watch out for injected imports

Breaking commonJS modules... webpack bundles commonjs as if it was esm? WHY?
https://github.com/webpack/webpack/issues/4039#issuecomment-274094298














[wp-treeshaking]: https://webpack.js.org/guides/tree-shaking/ "Webpack tree shaking"
[babel]: https://babeljs.io/ "BabelJS"
[babel-preset-env]: https://www.npmjs.com/package/@babel/preset-env "@babel/preset-env on npm"
[babel-repl-lodash]: https://babeljs.io/en/repl.html#?babili=false&browsers=&build=&builtIns=false&spec=true&loose=true&code_lz=JYWwDg9gTgLgBAbziAhmOBfOAzKERwDkANhACYoDOAFoQNwBQqYAFANoCMANAExcDMAXS5wAHnAC8APjFwA1HH4BKOkA&debug=false&forceAllTransforms=false&shippedProposals=false&circleciRepo=&evaluate=true&fileSize=false&timeTravel=false&sourceType=module&lineWrap=false&presets=latest%2Creact%2Cstage-2&prettier=false&targets=&version=6.26.0&envVersion= "Lodash import on babel repl"
