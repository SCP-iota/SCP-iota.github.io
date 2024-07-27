---
layout: blog_post
category: software
title: Not everything in Rust has to be an `Arc`
---

A common phenomenon among new Rust programmers is called "fighting the borrow checker": getting confused by lifetime and move restrictions and working against the compiler instead of with it. People learning Rust after already being familiar with other languages may not be sure how to deal with those compile-time checks, but others will try to systematically work around it...

# "C++-to-Rust" vs "Java-to-Rust"

There are two distinct mindsets when it comes to fighting the borrow checker: those who want more manual memory management and those who want less. People coming from languages like C and C++ who are used to explicit calls to `malloc`, `new`, and `free` might find Rust's insistence on stack allocation and standard library features odd. Meanwhile, people coming from garbage-collected languages like Java or Python don't want to have to worry about whether data exists on the heap or stack and when it gets deallocated.

# "Just work, darn it!"

Like it or not, the borrow checker exists for a reason. (Don't take my word for it - look up the many vulnerabilities caused by use-after-free edge cases!) If you're mostly familiar with automatic garbage collection, it'll be tempting to overuse the closest equivalent in Rust: [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) (or, if you don't need thread safety, it's single-threaded counterpart [`Rc`](https://doc.rust-lang.org/std/rc/struct.Rc.html)).

# If a tree falls in the forest...

Besides the verbosity of wrapping everything in a `Arc`, we also need to consider the differences between proper garbage collection and what `Arc` implements: reference counting. While a language like Java or Python keeps an internal graph of references to allocated data and periodically detects and frees unused allocations, reference counting merely keeps a count of how many references exist. When a `Arc` is passed around, it implicitly increments the reference count, and when it goes out of scope, it implicitly decrements the reference count and then automatically frees the memory if the count has reached zero.

That simplicity has its drawbacks, which is why garbage collection exists in the first place. If you have two reference-counted objects that both reference each other, their counts will always be at least one even if no other reference is still used by the running code. This kind of "island of isolation" can cause memory leaks as a program can create and abandon many of these circular reference structures throughout its runtime.

Some languages, like Swift, embrace reference counting and attempt to prevent islands of isolation by allowing code to keep "weak references," which don't count towards the reference count and must be checked before use. In this strategy, it's up to the programmer to use weak references to prevent memory leaks.

# "Behold, my `RefCell<Arc<Mutex<...>>>`-inator!"

...and then they complain that Rust is too verbose. It's because you're supposed to avoid using wrapper types when you don't have to, and structure your memory management around what your program needs instead of designing a one-size-fits-all workaround. It may seem like it shouldn't be necessary ("But I don't have to do that in Java!"), but there are performance gains to removing garbage collection, and the other alternative is☐

```
Segmentation fault (core dumped)
```