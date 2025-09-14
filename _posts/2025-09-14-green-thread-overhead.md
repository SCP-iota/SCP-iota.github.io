---
layout: blog_post
category: software
title: 'Green Thread Overhead - Async in a Trench Coat'
description: 'How do green threads work, and how do they compare to async functions?'
---

So, you just finished writing your handy higher-order utility function that does that one type of thing that pops up all over your codebase, and now you're ready to streamline everything. Only, there's a problem: some of the places you intend to use it require asynchronous operations, and now you have to choose between making your utility asynchronous, which will prevent it from being used in synchronous code, or avoiding using it for asynchronous tasks. You settle for creating a separate asynchronous version of the function, and for a brief moment, you feel like this dilemma is a synthetic limitation, and wonder if it could be solved if the language could "just let you call asynchronous functions synchronously." Then, you come to your senses. Almost.

## Why `async`?

Let's start by thinking about why we have `async` functions to begin with; what problem do they solve? The gist is that many kinds of operations, especially I/O, involve waiting until an external task has finished. During that time, other code could be running until the result is ready. That is what asynchronous code makes possible: "pausing" the flow of code in one task to allow other tasks to run, by internally breaking functions into separate chunks.

Consider, for example, the following asynchronous JavaScript function:

```js
async function getUsername() {
    const response = await fetch('/api/session')
    const json = await response.json()
    return json.username
}
```

It is functionally equivalent to this code, which does not use `async`:

```js
function getUsername() {
    return new Promise((resolve, reject) =>
        fetch('/api/session')
            .then(response =>
                response.json()
                    .then(json => resolve(json.username))
                    .catch(reject)
            )
            .catch(reject)
    )
}
```

Clearly, the former code is cleaner, and that is why the syntax sugar provided by `async` exists: to simplify working with tasks that cannot finish in a single flow of execution.

## Why Not Synchronous

This is where many people start to lose track of the point; the illusion of continuity created by the `async` syntax sugar leads some to forget the underlying mechanics and believe that the inability to call an asynchronous function from within a synchronous one is an artificial restriction.

"Why can't I do this?..."

```js
function getUserProfileURL() {
    const username = getUsername();
    return siteRoot + '/users/' + username
}
```

This textbook answer is that such code is incorrect because `getUsername` is an asynchronous function, and therefore cannot be called from `getUserProfileURL`, which is synchronous. But let's consider *why*: `getUsername` is asynchronous because it relies on an inherently asynchronous operation (a network request, which is in turn asynchronous because it must wait for I/O and a response.) Because of that, `getUsername` isn't so much a function that "gets the username" as it is a function that "*schedules the task of getting the username*." It *has* to be this way because there is no way to force the result to arrive immediately; the requester has to wait.

With that in mind, it becomes clear why a synchronous function like `getUserProfileURL` cannot call `getUsername`: `getUserProfileURL` does not "schedule a task," but rather is expected to do something and return a result immediately. It can't do that if it first has to wait on a sub-operation to finish externally.

## "Green Threads"

Some people point to languages that support green threads, such as Go, Dart, and (some versions of) Java, as evidence that it could be possible to design a language that allows calling an asynchronous operation from within a synchronous context.

Green-threaded languages work by fundamentally changing the way function execution happens; instead of a single program stack, a function's stack frame exists on one of several "green thread stacks," which are allocated by the execution environment that manages the green threads. Certain operations (for example, any I/O in Go) internally have an extra step at the end that gives the execution environment a chance to switch the active green thread if needed. A certain routine of the execution environment will be invoked that may determine that it's a good time to switch to a different green thread, and if it does, it will *store the address of program execution into a place associated with the current green thread and then jump to the previously stored program execution address of a different one.* This simulates "pausing" the current green thread and "resuming" another. Since the green threads have separate stacks, there is no interference from executing other code while one green thread is paused. This seems oddly familiar, though.

## `async` With Extra Steps

The above method of wrangling green threads is functionally equivalent to making every function `async`. Like the separate stacks, variables in `async` functions become properties of the internal state objects that get passed between executions of asynchronous tasks. Like the specific operations that trigger a chance to switch green threads, `async` functions use `await` as an indicator to yield continuity until the result is ready. And like storing execution addresses, registering callback functions provides a way for the environment to know what to call to resume execution of the task. Green threads aren't so much "threads" as they are the result of hiding asynchronicity behind *another* layer of syntax sugar: the language itself. That comes with a cost, too.

## Static Analysis, Security, and Platform Support

If you were to try to manually implement the kind of internal coordination done with green threads, you would likely run into a problem: it relies on arbitrary jumps. Of course, many languages support the use of jumps *within a function*, jumping to an arbitrary address in a different function is usually unsupported for a lot of reasons. One of those reasons is that static analysis algorithms, like those used in optimizers, security software, and even in your CPU itself to maximize the benefit of caching, rely on jump instructions only occurring in certain kinds of well-known patterns. For example, conditional blocks and loops can be detected from the way jump instructions are positioned relative to their target address.

If, however, an arbitrary jump instruction is found, it may become impossible for static analysis algorithms to understand the logic of the code. Even worse, the existence of *even one* arbitrary jump *anywhere in the code* can be enough to prevent some kinds of optimization and security guarantees even if there is no logical reason why the jump would interfere, simply because the algorithm can no longer be certain that the jump wouldn't send the execution flow to a position in the code that it could otherwise guarantee would only be accessed under specific conditions.

This problem also has a more concrete consequence than just disabling some optimizations: certain target architectures, notably WebAssembly, do not allow arbitrary jump instructions at all because they prevent static guarantees about the state of the stack. Since all WebAssembly functions are expressed in terms of values on the stack, while the compiled machine code usually maps these stack values into registers, the layout of the stack must be statically known and guaranteed. If it were possible to arbitrarily jump around in the code, unexpected code paths could unbalance the stack from its expected layout and cause fatal errors. Such code flow could even be used maliciously to break out of the sandbox. Since arbitrary jumps are not an option for these targets, languages that support green threads have to find workarounds. Often, the result is less efficient than using asynchronous functions.

## What Standard Library?

Then there's the issue of ensuring that those task-switching operations are called regularly or at key points. As mentioned, Go ensures that the execution environment gets a chance to switch green threads at every I/O operation, but that first requires a standard library where all I/O operations are provided with a call to that procedure. Which, of course, first requires a *standard library*. That's a problem for, say, embedded systems, where code is often compiled with only a minimal core library, and operations like I/O come from elsewhere.

## But That's None of My Business

That said, most of these concerns are related to statically compiled languages. One wouldn't expect to use static analysis on the JIT-compiled machine instructions of a Java program when the bytecode is already known. Nor would one expect to run their Go code on an embedded chip in their coffee maker. (Well, *maybe* not.) But it still seems important to consider the caveats to different execution models and how that could affect what environments they may or may not be ideal for.
