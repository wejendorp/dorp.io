---
title: "Express HMR with webpack"
category: programming
tags: [webpack]
---

Finally got around to setting up a working express hot module reloading config.

When, a year ago, I started replacing a custom `Module` implementation in our
express servers with plain node modules in the name of simplicity, my colleagues
started complaining about no longer having server side reloading.
But with this, developing on the server is back to its former glory!

The purpose is to have only part of an express server work in this way, so that
a module being required can update itself in place.

We'll focus on getting a single module, and all it's subdependencies to reload
during development. Specifically, how to dynamically discover [express.Router](http://expressjs.com/en/api.html#express.router) modules and mount them with
reloading on an existing "normal" express server.


If you're looking for a way to run your entire server through webpack, take a look at
[Hot reload all the things!](https://hackernoon.com/hot-reload-all-the-things-ec0fed8ab0)

# Webpack config

Let's start out with a minimal `webpack.config.server.js`, that we'll use as an
example. We want to run everything through [babel](https://babeljs.io/), to get
that awesome [`es2018`](http://2ality.com/2017/02/ecmascript-2018.html) feature
support into node.

```js
// webpack/webpack.config.server.js

const webpack = require('webpack');
const path = require('path');
const nodeExternals = require('webpack-node-externals');

module.exports = {
	entry: ['webpack/hot/poll?1000', './src/mount.webpack'],
	target: 'node',
	output: {
		path: path.join(__dirname, '.build'),
		filename: 'server.js'
		// expose main method as:
		library: 'mount',
    // Build it as a commonjs library so we can include it
		libraryTarget: 'commonjs'
  },
	// Don't bundle node_modules, they'll be available at runtime
	externals: [
		nodeExternals({
			// Include the hot reload polling code in the bundle though
			whitelist: ['webpack/hot/poll?1000']
		})
	],
	module: {
		rules: [
			{
				test: /\.js?$/,
				use: 'babel-loader',
				exclude: /node_modules/
			},
			{
				test: /\.json$/,
				use: 'json-loader'
			}
		]
	},
	plugins: [
		new webpack.NamedModulesPlugin(),
		new webpack.HotModuleReplacementPlugin(),
		new webpack.NoEmitOnErrorsPlugin()
	]
};
```

# Autodiscovery with require contexts

The auto discovery loader could look something like this, before we add webpack:

```js
// src/mount.js

const glob = require('glob');
const path = require('path');

module.exports = app => {
	// Automatically mount all *.express.js files:
	glob('src/routes/*/*.express.js', function(err, files) {
		files.forEach(file => {
			const entrypoint = require(file);
			// Handle ES6 default exports:
			const entrypointRouter = entrypoint.default || entrypoint;
			const mountAt = path.basename(path.dirname(file));
			app.use(`/${mountAt}`, entrypointRouter);
		});
	});
};
```

To be included in our express server like so:

```js
// index.js

const express = require('express');
const app = express();

require('./.build/server').mount(app);

app.listen(3000);
```

But how can we keep this auto discovery, when we add webpack?
`require(file)` can't very well be statically analyzed, and trying to build it
will generate errors instead.

That's where [`require.context`](https://webpack.js.org/guides/dependency-management/#require-context)
comes in. `require.context` is like webpacks own globber, with the added
knowledge of modules and bundling. It looks like this:

```js
require.context(directory, (useSubdirectories = false), (regExp = /^\.\//));
```

```js
// src/mount.webpack.js

const glob = require('glob');
const path = require('path');

module.exports = app => {
	// Automatically mount all *.express.js files:
	const contextRequire = require.context('src/routes', true, /\.express\.js$/);
	const files = contextRequire.keys();

	files.forEach(file => {
		// Use the new context aware require to include the module:
		const entrypoint = contextRequire(file);
		// Handle ES6 default exports:
		const entrypointRouter = entrypoint.default || entrypoint;
		const mountAt = path.basename(path.dirname(file));
		app.use(`/${mountAt}`, entrypointRouter);
	});
};
```

# HMR, require.context and express

In order to reload the dynamic context, we must `hot.accept` it by id, and then
re-evaluate the context. Then we can swap out the route handler in express, and
see our changes.

```js
let context = require.context('src/routes', true, /\.express\.js$/);
if (module.hot) {
	module.hot.accept(context.id, function() {
		// Re-evaluate context to get updates:
		context = require.context('src/routes', true, /\.express\.js$/);
		// Do something with the updated library module...
	});
}
```

In our server, we can use the callback to re-evaluate our routes, and mount a
new Router instance into express:

```js
// src/mount.webpack.js
const { Router } = require('express');

module.exports = app => {
	let expressRouter;
	let buildPromise;
	let contextRequire;

	const getRouter = () => {
		contextRequire = require.context('src/routes', true, /\.express\.js$/);
		const files = contextRequire.keys();
		const newRouter = Router();

		files.forEach(file => {
			// Use the new context aware require to include the module:
			const entrypoint = contextRequire(file);
			// Handle ES6 default exports:
			const entrypointRouter = entrypoint.default || entrypoint;
			const mountAt = path.basename(path.dirname(file));
			app.use(`/${mountAt}`, entrypointRouter);
		});
	};
	// Initial load:
	getRouter();

	// Hot reload the context
	if (module.hot) {
		module.hot.accept(contextRequire.id, getRouter);
	}

	app.use((req, res, next) => {
		// A swappable route handler for HMR updates
		expressRouter(req, res, next);
	});
};
```

Now, if we try and make any changes to a matching router, we should see
something like

```
[HMR] Updated modules:
[HMR]  - ./src/routes/auth/login.express.js
[HMR]  - ./src/routes recursive \.express\.js$
[HMR] Update applied.
```

And the router should have been automatically swapped out.

# A simple dev-server middleware

Having been spoiled by `webpack-dev-server` and `webpack-dev-middleware`, running
`webpack --watch` on the side doesn't feel quite right. Luckily, we can use the
[Node.js API](https://webpack.js.org/api/node/#watching) of webpack to do something similar.

That means adding an automatic wait for build, and compiling during startup.

```js
// index.js
// "webpack-dev-server" minimal example

const express = require('express');
const app = express();

const isDev = process.env.NODE_ENV !== 'production';

if (isDev) {
	const webpack = require('webpack');
	const webpackConfig = require('./webpack/webpack.config.server.js');

	const buildPromise = new Promise(resolve => {
		const compiler = webpack(webpackConfig);
		compiler.watch({ filename: paths.appSrc }, function(err, stats) {
			if (err) {
				console.error(err);
			}
			resolve();
		});
	}).then(() => {
		// Mount the routes when ready (and only once via promise semantics):
		require('./.build/server').mount(app);
	});

	// waitForBuild middleware to avoid confusing 404s
	app.use((req, res, next) => {
		buildPromise.then(next, next);
	});
} else {
	// In production mode, load it as before:
	require('./.build/server').mount(app);
}

app.listen(3000);
```

# Source-maps for sanity and production

To keep sane with transpiled code, we want source-maps in the node process itself.
This is easily accomplished by making sure webpack config is set to
`devtool: 'inline-source-map'`, and the following snippet injected before
importing the bundled code:

```js
require('source-map-support').install({
	// hookRequire for inline-sourcemaps support:
	hookRequire: true
});
```

# Conclusion

We had to jump through a few hoops, but the server is happily compiling as
needed and hot reloading during development.

Before starting in production, use webpack CLI to generate the mount file:

```bash
$ webpack --config webpack/webpack.config.server.js
```

Have fun iterating even faster on your express modules!
