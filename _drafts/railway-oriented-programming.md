---
title: "Railway Oriented Programming in Javascript"
category: programming
tags: [fp, promises]
---
*This post is entirely based on the post and talk [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/)
by [Scott Wlaschin](https://twitter.com/scottwlaschin).

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
whether intended or accidental. So be careful about always adding a `catch` on the end of your tracks,
or the payload will never be seen again! Well... this is not quite true in modern browsers, Chrome
recently started warning about [unhandled promise rejections](http://2ality.com/2016/04/unhandled-rejections.html),
but in other environments the error might just get swallowed.

I recommend using [eslint-plugin-promise](https://github.com/xjamundx/eslint-plugin-promise) to get
some static analysis help for those cases.

# Functions as little railways
Consider each function a separate piece of railway.
<img src="/assets/images/railway-2.png" alt="Rails!" style="width: 250px;"/>

Representations of errors...

# References

* Server side example: BOOM

Examples of
