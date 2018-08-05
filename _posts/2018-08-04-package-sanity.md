---
title: Package sanity checks
category: programming
tags: [node]
---

There are already plenty of ways developing packages for node can be confusing,
especially in an environment where not every contributor is an avid npm package
maintainer.

Some common mistakes are easily prevented, and avoiding them will save much time
and frustration.

* Is this the correct version of `node`?
* Are the dependencies installed correctly, with the right versions?

These issues can be solved once and for all with some `npm` / shell magic.


# Run the right node
Likelely nothing will work if we end up running the wrong version of node.
Whether that is a too new or too old version, it might cause all kinds of
unpredictable behavior.
To make sure we run the right version for a package we have a few options.

## npm engines
The first official tool at our disposal is the [package.json engines](https://docs.npmjs.com/files/package.json#engines) property.
It advises consumers to use a spefic version of `node` with the `engines` when the package is installed.

```json
{
  "name": "my-package",
  "engines": {
    "node": "^8.11.0",
    "npm": "^6"
  }
}
```

This will generate a warning if the package is installed with an unsupported version,
according to the [semver ranges](https://docs.npmjs.com/misc/semver#ranges).
Here we are a bit conservative and only want node versions greater than `8.11`
but less than 9.

This is not quite enough for enforcing a version when working on the package itself,
but it's our only option for telling consumers about potential compatiblity issues.

## Install node locally
An option if the repository requires a specific node version for development, or
if the repo is not a package to be published. We can install an exact version of
node, which will be installed like an npm package, and added to the path for
npm run.

```bash
$ npm install nodejs@8.11.3 --save-exact --save-dev
```

That way we run a small test with `node -v` and end up with a `package.json` like this:

```json
{
  "name": "my-package",
  "scripts": {
    "start": "node -v"
  },
  "devDependencies": {
    "node": "8.11.3"
  }
}
```

```bash
$ npm start
v8.11.3
```

## Using [nvm](https://nvm.sh)
We can add an `.nvmrc` file to the root of the repo to tell developers which version
to use. Whenever we see such a file, it should contain the recommended node version,
and running `nvm use` will make sure we're running the way the author intended.

```bash
$ echo "8" > .nvmrc
$ nvm use
Found '[...]/.nvmrc' with version <8>
Now using node v8.11.3 (npm v6.1.0)
```

If we embrace the `.nvmrc` as a standard, we can automate the version switching
by adding some shell magic, to look for it and run `nvm use` if it exists.

[nvm shell integration](https://github.com/creationix/nvm/tree/17c33fd9aeb42261d38fea676246700898ea2855#deeper-shell-integration)

# Avoid stale modules: `check-versions`

Granted, this is probably a non-issue on smaller repositories, but with enough
developers working on the same code this is bound to happen pretty often:
After pulling the latest version, or changing branches to review some changes,
something isn't quite right. The console is red and the messages are completely
incomprehensible. Whoops, the `package.json` has changed, perhaps with some incompatible
version of `angular`, but the server unknowingly churns out a broken bundle.

Avoid wasting hours trying to debug an inconsistent state of the world,
or from losing a potential contributor by adding
[check-dependencies](https://www.npmjs.com/package/check-dependencies) and catching
the problem early.

We can run `check-dependencies` without an explicit dependency with `npx`.
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
  "name": "my-package",
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

```bash
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
