---
title: Package sanity checks
category: programming
tags: [nodejs]
---

There are a lot of ways in which developing packages for node can be confusing,
especially in an environment where not every contributor is an avid npm package
maintainer.

Some common mistakes are easily prevented, and avoiding them will save much time
and frustration.

* Is this the correct version of `node` for this project?
* Are the dependencies installed correctly, with the right versions?

These issues can be solved once and for all with some `npm` / shell magic.


# Run the right node
Likelely nothing will work if we end up running the wrong version of node.
Whether that is a too new or too old version, it might cause all kinds of
unpredictable behavior.

If the project is using newer node features, running `npm start` might look like
this:

```bash
(function (exports, require, module, __filename, __dirname) { async function foo() {
                                                                    ^^^^^^^^

SyntaxError: Unexpected token function
    at exports.runInThisContext (vm.js:53:16)
    at Module._compile (module.js:373:25)
    at Object.Module._extensions..js (module.js:416:10)
    at Module.load (module.js:343:32)
    at Function.Module._load (module.js:300:12)
    at Function.Module.runMain (module.js:441:10)
    at startup (node.js:140:18)
    at node.js:1043:3

```

Instead of leaving people to guess what went wrong, we can try and detect this,
giving a much friendlier error.
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

```
npm WARN engine my-package@1.0.0: wanted: {"node":"^8.11.0"} (current: {"node":"8.9.4","npm":"5.6.0"})
```

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

Basically any run-script we add will use the `node` binary from `./node_modules/.bin`
instead of the one from the system.

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

The official repo has examples of [nvm shell integration](https://github.com/creationix/nvm/tree/17c33fd9aeb42261d38fea676246700898ea2855#deeper-shell-integration)

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

```bash
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

There's a slight caveat though. If we start using `npm link`ed packages to fix an issue in a
subdependency, the version in `package.json` still needs to match, or our package will
do as we just asked, and refuse to start.

```bash
$ npm start

> Apps@2017.0.0 prestart [...]/Apps
> check-dependencies

@tradeshift/v4: installed: 0.0.0-semantically-released, expected: 12.9.0
Invoke npm install to install missing packages
npm ERR! code ELIFECYCLE
npm ERR! errno 1
```

Finally, the node api of `check-dependencies` can be used to customize the
behavior, and ignore certain types of mismatches, such as symlinked packages.

But by either disabling the `prestart` script or specifying the version manually
while testing, we can test without the check getting in the wahy.
