---
title: "Async javascript"
category: programming
---

Synchronous code is easy to read, easy to write. Everything is simple until we need to
fetch something from a database, or make an HTTP request. Pausing the execution to get
a value will make the browser unresponsive, or for server side code, will make the
server not respond to new requests.

This is because of the single-threaded nature of Javascript. So we need to be clever
about writing our code in a way that we do slow operations in the background and continue
when the data is ready.

There are a

---

# Callbacks

Pyramid of doom


If we could write our code as

```js
function searchByAuthor(author) {
	const authors = fetch('/authors', author);
	const posts = postsByAuthor(authors);
	return posts;
}
```

It is also incredibly impractical

Blocking the

The "new" thing? well not really. fetch api example

Promises are simple.
depends on the language.. java promises 130 methods?


F# Rail-way programming?


Error Handling - when
an easy rule of thumb: Catch-or-return

unhandled rejections in node 6+ - deprecated, becoming fatal

Async javascript - promises or callbacks

# Node-style callbacks
Node has a callback convention that encourages error handling at every step.

```js

```

# Promises
Wrapping "node-style" callbacks.

Getting help: mz module

# ES2016 async functions

...

# Useful links
There's plenty of awesome reading on the topic of promises and async javascript.

* https://developers.google.com/web/fundamentals/getting-started/primers/promises
