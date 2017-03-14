---
title: "Abstractions are mental overhead"
category: programming
---

It's incredible how quickly everything moves in our field. Nothing makes this more clear than
looking through our own old code, or in this example old posts right here on the blog.

The quest for not repeating anything makes us do crazy hacks and abstractions, just to avoid having
those very error prone `background-color: red;` "duplicated" between two files.

Most abstractions seem beneign at conception, but tend to mutate with time. The more abstractions
it has, the more cracks will appear when you need to move or change an interface between classes/modules.
When a feature needs adding, and falls into the void, who knows where it ends up? Sometimes `math.random()` might
as well decide, since "anywhere" will seem perfect when deadlines loom.

Even when you perfectly understand the entire codebase, figuring out which classes to change can be hard.
Now pass this code on to the next guy, who has none of the context. How many concepts will they
have to learn in order to work in the framework and vocabulary that you have created without breaking it?

Avoid leading your colleagues deeper into the rabbit hole of random abstractions by keeping the APIs simple.
Use simple or common datatypes where possible.

Avoid [over engineering your APIs](https://www.quora.com/Was-object-oriented-programming-a-failure/answer/Wouter-van-Oortmerssen?share=1)
to please the OOP gods.

## Don't try to be clever with indirections
Tunnel vision comes in many forms. Trying to decouple your code is another example. In this example,
a post from MSDN called [Active events](https://msdn.microsoft.com/en-us/magazine/mt795187).
Basically, this article describes a pattern to avoid having any explicit dependencies in your code,
in the attempt to make it easier to replace parts of it with another DLL.

This hits a nerve with me, as implicit dependencies are at the core of many bugs I've seen lately.
You might be able to replace the `AuthenticationProvider` from any file in the project now, but is
this a fox in sheeps clothes? I think it is, since you have most certainly just confused your IDE,
and possibly also your colleagues. If the code that runs when invoking `dispatch("auth-please")` can
live anywhere, you might as well have just done  address space randomization on you project files.
You have decoupled the code, your IDE and any possibility of doing explorative discovery to learn
how it works.

Working in a system of implicit dependencies can be extremely draining, since the effects of change
are hidden behind layers of indirection and abstraction, and you get zero help from standard tooling.
The less your tools can help you, the more mental capacity is needed to juggle the parts and their
relationships. Even more so, to start changing these relationships, when requirements change.
