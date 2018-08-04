---
title: Package sanity checks
---

There are already plenty of ways developing packages for node can be confusing,
especially in an environment where not every contributor is an avid npm package
maintainer.

Some common mistakes are easily prevented, and avoiding them will save much time
and frustration.

* Is this the right version of `node`?
* Are our packages installed correctly?

These issues can be solved once and for all with some `npm` magic.

# Run the right node
WIP:

* npm engines
* nvm
* npm i node
* react-scripts example


# Avoid stale modules: `check-versions`

Granted, this is probably a non-issue on smaller repositories, but with enough
developers working on the same code this is bound to happen pretty often:
After pulling the latest version, or changing branches to review some changes,
something isn't quite right. The console is red and the messages are completely
incomprehensible. Whoops, the `package.json` has changed, perhaps with some incompatible
version of `angular`, but the server unknowingly churns out a broken bundle.

Avoid wasting hours trying to debug an inconsistent state of the world,
or from losing a potential contributor by adding `check-dependencies`, and catching
the problem early.

We can also run `check-dependencies` without an explicit dependency with `npx`.
To check the state of the world, for any package.

```
$ npx check-dependencies
@tradeshift/document-components: installed: 4.14.0, expected: 4.15.0
@tradeshift/v4: installed: 12.8.1, expected: 12.9.0
Invoke npm install to install missing packages
```

For instance by putting it in `prestart`, the program will be unable to start when the
contents of `node_modules` doesn't match `package.json`.

```json
{
  "scripts": {
    "prestart": "check-dependencies",
    "start": "node ./index.js"
  },
  "devDependencies": {
    "check-dependencies": "^1.1.0"
  }
}
```

There's a slight caveat though. If we start using linked packages to fix an issue in a
subdependency, the version in `package.json` still has to match, or our package will
do as we asked, and refuse to start.

```
$ npm start

> Apps@2017.0.0 prestart /Users/jacob/dev/Tradeshift/Apps
> check-dependencies

@tradeshift/v4: installed: 0.0.0-semantically-released, expected: 12.9.0
Invoke npm install to install missing packages
npm ERR! code ELIFECYCLE
npm ERR! errno 1
```

Can't win them all... But by either disabling the `prestart` script
or specifying the verison manually, this can be overcome.
