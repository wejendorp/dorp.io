---
title: 'Node module reloading'
category: programming
tags: [node]
---

Working with node servers is part of the job.
While iterating on such servers, it's common to go for a solution involving
automatic restarts using something like `nodemon`.

```bash
npx nodemon --exec "npm start" --watch src
```

This will automatically restart the server whenever there are changes to the
src folder. But I think we can do better than that!

The restart can be expensive, depending on the kinds of initialization that's
involved. What if we just make node reevaluate only the parts that changed?

<!-- cut -->

# Forget that module

If we mess a little with the module (require) cache of node, we can break the
normal singleton handling of modules, and run it again, with any changes we might
have added in the meantime.

```js
// forgetModule.js
// const { flatMap } = require('lodash/fp');
const flatMap = fn => xs => xs.reduce((agg, x) => agg.concat(fn(x)), []);

module.exports = function forgetModule(filename) {
	// Clear the module cache, load any updates from filesystem:
	const cached = require.cache[require.resolve(filename)] || {};
	const getModules = flatMap(m => [m.id].concat(getModules(m.children)));
	getModules([cached])
		.filter(s => s && !s.match(/\/node_modules\//)) // but leave node modules
		.forEach(fn => {
			delete require.cache[fn];
		});
};
```

With that, we will get the first-load behavior (evaluation) on any following
`require`:

```js
require('./log'); // console.log('hi');
require('./log'); // - cached, nothing happens.
forgetModule('./log');
require('./log'); // console.log('hi');
```

The module will be read from disk and evaluated fresh. Success!

# Swappable routing

Forgetting modules is all well and good, but how do we make something like
`express` swap out it's routes? There is no API to interfere with existing
routes in the middleware stack.

We can instead add a route function (which will be locked in the middleware stack)
that is dynamic, and all we have to do then is swap out the handler function in
there.

```js
// index.js
const express = require('express');
const forgetModule = require('./forgetModule');

const app = express();

// A non-swappable router:
app.use(require('./fixed-routes'));

// Swappable route handler
app.use((req, res, next) => {
	// To keep it simple, reload the routes on every request:
	if (process.env.NODE_ENV !== 'production') {
		forgetModule('./routes');
	}
	// Anywhere we want to support hot-swapping, we need to require in place
	require('./routes')(req, res, next);
});

app.listen(3000);
```

```js
// routes.js
const Router = require('express').Router;
const router = Router();

router.get('/', (req, res) => {
	res.send('Hello world');
});

module.exports = router;
```

# Watch and forget. Reload on change

If the simple always reload solution seems a little too coarse, we can always
combine it with a file watcher, still without having to restart the whole process:

```js
// watch.js
const chokidar = require('chokidar');

chokidar.watch('./src').on('all', (event, path) => {
	forgetModule(path);
});
```

If the inline requires are set up, we can use `require('./watch')`, and reload
on change instead of on every request.

_Caveat:_ A watcher like this will only work if you have an inline require to the file
that is being removed. I.e. changing `routes.js` from above will work, but
changing `routes/auth.js`, required from `routes.js` will not since `routes` is
still cached. One possible solution is to use the watcher to invalidate, but
still invalidate the root, i.e. `forgetModule('./routes')`.

Another solution could be to traverse upwards in the module tree when invalidating,
until reaching a "stop" file, such as the routes file. This would give a more
fine grained invalidation, in cases where invalidating all our routes is causes
problems.
