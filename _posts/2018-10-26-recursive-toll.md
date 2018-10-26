---
title: "The recursive toll"
subtitle: "How recursion can kill performance"
category: programming
---

With the rise of functional programming in Javascript, it's increasingly likely
that you'll have come across, or written recursive functions. They allow us to
express the solution to a big problem in terms of solving "this one step,
plus the solution to the rest".

If we express that in a more functional notation, it's akin to `fn(x :: xs) = x + fn(xs)`.
This post serves to warn about the limitations of this type of function, and to
suggest an alternative for when the situation requires us to exceed those limitations.

# Recursive functions and the call stack

Say we want to express the sum of a list of numbers, we can express it as the
value of the first element, plus the sum of the rest: `list[0] + sum(list.slice(1))`.
We also need to handle the empty list, to create our base case and avoid infinite
recursion. Expressed as a ternary (inline if statement), this becomes:

```js
const sum = list => list.length ? list[0] + sum(list.slice(1)) : 0;
```

If we try it out with a small list of numbers, everything looks fine.

```js
sum([1, 1, 2, 2]); // 6
```

If suddenly the guy producing the numbers had a really productive day, and gave us
a list of 1.000.000 numbers, we have a problem:

```js
// create list of one million 1s:
const oneMillionInSingles = Array.from(new Array(1000000), () => 1);
sum(oneMillionInSingles);
// Uncaught RangeError: Maximum call stack size exceeded
```

It comes down to the call stack, since every time we call a function, the engine
needs to keep track of where we were, so it can resume from there when the function
returns. By doing this in a way that is dependent on the input size, we naturally
limit how big problems we can solve with that function without encountering the
infamous `Uncaught RangeError: Maximum call stack size exceeded`.

The limit might vary depending on the engine, browser, node version, but we should
be grateful that it's trying to save us from what is by any reasonable measure,
an infinite recursion.

So how can we handle bigger inputs without blowing the stack?

# Beating the stack limit with a loop
What if we just used a loop? I know, I know, we will lose all those cool points,
and our street cred might never fully recover. But sometimes it's better to be
correct than cool. Stay with me.

The `sum` function can be written with a loop.

```js
const boringSum = list => {
  let acc = 0;
  for(let i = 0; i < list.length; i++) {
    acc += list[i];
  }
  return acc;
};
```

It also comes with a significant performance boost, since we are not creating all
those stack frames, and can sometimes make the difference between a solution
that is too slow, and one that is fast enough™.

# Tail call optimization
This is basically what the compiler will do in languages that have proper tail
call optimization. But only if the code is written in a form that allows it.
The rule of thumb is that if the expression cannot be evaluated before recursing,
the compiler cannot help you.

This is commonly done by introducing an accumulator. Basically we keep a running sum,
such that instead of creating a call stack with `sum([1, 2, 3, 4])` of
`1 + (2 + (3 + (4)))`, we can evaluate the plus immediately.

This allows the runtime to swap this stack frame with the new function call.
There is no longer a need to remember every previous frame in order to get the
final result.

```js
const tailcallSum = (list, acc) => list.length ? sum(list.slice(1), list[0] + acc) : acc;
```

**But...** NodeJS had tail call optimization for a while
[(`v6.2` to `v7.10`)][stackoverflow-tco], but due to limitations in the
underlying V8 engine, it is no longer available.

As long as we write Javascript, we have to consider the limits
to our recursive functions. Perhaps some day it will return.

When that day comes, because our code will live long enough to become
it'd be awesome for our functions to be ready, and get a free
performance boost!

# Conclusion and benchmarks
This is a classic case of don't do this at home. Or don't base your decisions on
micro benchmarks, since a real world use case will involve all kinds of
compiler magic, that might interfere with the characteristics of a small snippet
of code, in order to better optimize on a higher level, or because the optimizing
compiler has deoptimized your code due to exceptions.

With those disclaimers out of the way, let's look at the numbers:

...

[stackoverflow-tco]: https://stackoverflow.com/questions/23260390/node-js-tail-call-optimization-possible-or-not#30369729
