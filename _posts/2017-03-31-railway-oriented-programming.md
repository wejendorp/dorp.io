---
title: "Railway Oriented Programming in Javascript"
category: programming
tags: [fp, promises]
---
A mental model for promise chains and error handling, adapted to javascript promises.
*This post is entirely based on the post and talk [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/)
by [Scott Wlaschin](https://twitter.com/scottwlaschin).*

I am quite fascinated with our tendency to focus on the happy code path, which often leaves
error handling as an afterthought. It always bothered me, and with the recent popularization of
functional programming techniques I think I have finally found a mental model for error handling
that I like. It's the [either monad](https://www.schoolofhaskell.com/school/starting-with-haskell/basics-of-haskell/10_Error_Handling#the-either-monad)!

As it turns out, promises in Javascript are quite analogous to the Either monad. If you have no idea
what that means, I hope the Railway analogy will help you as it did me.

Most functions that are doing external calls like HTTP requests, file system operations or runtime
validation will have at least two different outputs: a success case and an error case.

```js
function generateProfit() {
	collectUnderpants()
		.then(undefined)
		.then(profit => {
			// Get the result (success case)
			console.log('Profit!');
		})
		.catch(error => {
			// And of course handle the errors:
			console.error('This shouldn\'t happen');
		});
}
```

By going with the railway analogy, we have an error track and a success track, or "the happy path".

![alt](/assets/images/railway-1.png)
Image sourced from [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/).

With promises, I suppose the error track also works as a guard rail. No errors will escape the promise,
whether intended or accidental such as type errors.
So be careful about always adding a `catch` on the end of your tracks, or the payload may never be seen again!
Well... this is not quite true in modern browsers, Chrome and Node recently started warning about
[unhandled promise rejections](http://2ality.com/2016/04/unhandled-rejections.html),
but in other environments the error might just get swallowed.

I like using [eslint-plugin-promise](https://github.com/xjamundx/eslint-plugin-promise) to get
some static analysis help for those cases.

# An Express(JS) Railway example!
Consider each function a separate piece of railway. Validate will be the starting point.


<img src="/assets/images/railway-2.png" alt="Rails!" style="width: 250px;"/>

Consider this example signup route for an ExpressJS application:

```js
function signup(req, res, next) {
	validate(req)
		.then(createUserFlow)
		.then(() => {
			res.sendStatus(200); // success!
		})
		// Main error track
		.catch(error => {
			if (error.error) { // debug info
				console.error(error.error);
			}
			res.status(error.status || 500).send(error.message); // nice user error
		});
}
```

Which is composed of two steps; different variations of promise functions with error handling.

Validate is a synchronous function, but returns a promise to get our railway started. If the request
is valid, the request follows the happy path to continue signup.

```js
function validate(req) {
	if (req.body.email !== 'foo@awesomesauce.com') {
		return Promise.reject(statusError(400, 'Invalid email'));
	}
	return Promise.resolve(req);
}
```

Signup is more complicated, we can get all kinds of errors back from the database, and all kinds of
errors back from the smtp server. To the user they are all the same, so we can combine these two
steps into one with a more general error, and keep the details for our internal logger.

```js
function updateDB(req) {
	return User.create({email: req.body.email})
		.catch(error =>
			// Assume it's a database error
			// Let's map this to an HTTP 500
			Promise.reject(statusError(500, 'Unable to create user', error)
		);
}

function sendEmail(user) {
	return smtp.sendMail(user.email, 'Welcome to awesomesauce!')
		.catch(error =>
			// SMTP error
			Promise.reject(statusError(500, 'Unable to send email', error)
		);
}

function createUserFlow(req) {
	return updateDB(req)
		.then(sendEmail)
		// Error path
		.catch(error =>
			// What does this failure mean to the user?
			Promise.reject(statusError(500, 'Unable to complete signup', error))
		);
}
```

`Promise.then` works to connect the happy path of one function with another. We could write a
compose function for promise functions like so:

```js
function compose(...fns) {
	return x => fns.reduce(
		(res, f) => res.then(f),
		Promise.resolve(x)
	);
}
```

which would let us define the flows as compositions:

```js
// req -> bool
const createUserFlow = compose(updateDB, sendEmail);
// req -> bool
const requestHandler = compose(validate, createUserFlow);
```

Validate is `req -> req`, so we can compose (or connect in rail terms) to get one operation.
Caveats: `Promise.catch` will ... catch the rejection, unless you explicitly reject the promise
again.

To be fair this is not entirely a 1:1 mapping to the railway concept. We have only really done a
happy-path connector. There is not really a nice way to write the function that handles both cases
in javascript, since F# as a proper functional language relies on types and pattern matching.
If you feel like this is not enough, you can use regular composition and write functions that
take promises in, and return promises. That way you can handle both rejection and resolve cases.

```js
function composeAll(...fns) {
	return x => fns.reduce(
		(res, f) => f(res),
		Promise.resolve(x)
	);
}
const toReject = p => p.then(x => Promise.reject(x));
const toSuccess = p => p.catch(x => x);
const tap = fn => p => p.then(x => { fn(x); return x; });

// Swap back and forth
composeAll(
	toSuccess,
	toReject,
	tap(console.log), // skipped
	toSuccess,
	tap(console.log) // foo
)(Promise.reject('foo'))
```
