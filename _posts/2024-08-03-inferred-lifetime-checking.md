---
layout: blog_post
category: software
title: Inferred Lifetime Management: Could we skip the garbage collector *and* the verbosity?
---

I like Rust; it's possibly my favorite programming language. (It's not a cult I swear!) One thing I *really* like about Rust is its no-cost memory management. Most modern languages, like C#, Python, JavaScript, Swift, etc. implement memory management at runtime using methods like garbage collection or reference counting, but those methods come at a cost to performance. Lower-level languages like C and C++ leave memory management largely to the programmer, inevitably leading to crashes, bugs, and security vulnerabilities. Rust, however, prefers to implement memory safety at *compile-time*, giving us the closest possible solution to the best of both worlds. It has a caveat, though...

# Compile-Time Lifetimes

A major part of Rust's memory safety is that the compiler has an idea of when each value is created and when it is dropped, information called a *lifetime*. This lets it detect if a reference is passed in a way that might cause a use-after-free issue: the compiler will never let you (safely) pass a reference to something that expects a lifetime longer that of the original value. For example, you can't return a reference to a local value:

```rust
fn get_number_reference() -> &i32 {
    let n = 42;
    &n
}
```

It will inform you that it expects a lifetime specifier. That is, you must tell it the lifetime of the value you're returning. It will *not* assume it to be the lifetime of the function, since it makes no sense to return a reference value that is already gone by the time the function has returned.

# Explicit Lifetimes

Lifetimes are not just an implementation detail of the compiler, though. There are times you may need to explicitly specify a lifetime on a reference. For example, if you wanted to keep trying futilely to get Rust to let you return a reference to a local value, you might try this:

```rust
fn get_number_reference<'l>() -> &'l i32 {
    let n = 42;
    &n
}
```

In this example, the function has a lifetime parameter: `'l`. (Lifetime names in Rust start with `'`.) It's a generic parameter because it only exists at compile type, similar to a type parameter. In this case, the caller will not need to pass an explicit lifetime parameter, and can instead simply do `get_number_reference()` as if there wasn't one, because Rust will infer the lifetime to be the same as the function's lifetime unless the caller specifies otherwise.

The problem still remains, though: Rust's borrow checker will detect that you're passing a reference outside it's lifetime, and will not compile.

This shows a handy feature of the syntax, though: even though the function has an explicit lifetime parameter, the caller doesn't have to explicitly pass one unless it wants to use a lifetime other than the function's own lifetime. Not all used of lifetimes in Rust are that simple, though - consider a struct that holds a reference:

```rust
struct MyStruct {
    num_ref: &i32
}
```

Rust will tell you it's missing a lifetime specifier. This is where Rust starts getting strict: you *must* specify a lifetime for references as struct members. It can be fixed like this:

```rust
struct MyStruct<'l> {
    num_ref: &'l i32
}
```

In this case, `MyStruct` has a lifetime parameter and uses it as the lifetime for its `num_ref` member. As with functions, the user of the struct does not need to explicitly specify a lifetime because the compiler will assume it to be the same as the lifetime of the instance - with an exception: if another struct has a member that is an instance of a struct that takes a lifetime parameter, the outer struct must *also* use a lifetime parameter. In other words, the issue propagates:

```rust
struct OuterStruct<'l> {
    inner: &'l MyStruct<'l>
}
```

The `OuterStruct` must take a lifetime parameter and use it as both the lifetime of the reference and the explicit lifetime parameter of the `MyStruct` type. This is verbose; we can do better, but only if we can make some assumptions...

# Function Programming and the Copying Problem

While Rust is not a functional programming language (in the sense of pure functions) because it allows side effects and non-deterministic results, languages like Scala, F#, Haskell, and Lisp are specifically designed to be centered around pure functions. One benefit this paradigm gives us is that we don't have to worry about values being mutated - once a value is created, whether it's a primitive type or a complex type like a struct, array, or map, none of its contents will change. Instead, to simulate changing contents, a *new* value will be created that reflects the expected changes.

Right off the bat there are performance concerns: if, to add an element to an array, I have to create a whole new array that copies the previous array's elements and also has the new element, won't there be a significant cost to all that copying? It depends - if you did that in something like Python, then yes, but Python wasn't designed specifically for functional programming. Languages like Lisp store collections different to optimize such operations, often using a binary tree or similar structure to make it possible to add new elements simply by wrapping the original collection in a new layer. It's much more efficient, but there's still a cost, including a cost to *reading* from such a structure.

# Optimization-First Functional Paradigm

I propose a design for a functional programming memory model that uses a more traditional method of storing collections like arrays and maps while also avoiding the cost of copying whenever possible and preserving the O(1) nature of read access.

In this model, every type will be one of these:

* **Value** - An owned value type
* **Static Reference** - A reference to static, immutable data that exists in memory for the entirety of the program's execution
* **Shared Reference** - A possibly shared reference to a value
* **Exclusive Reference** - An unshared reference to a value
* **Heap Reference** - An reference to a heap-allocated value, from the perspective of the owner of the value
* **Passthrough Return** - A designation of return type that the compiler gives to a function that returns a mutation of one of its arguments. It internally indicates that the operation can be done in-place. *Only* for return types.

Imagine, in pseudocode, a pure function like this:

```
fn my_function(): (int, int) {
    let my_array = [2, 4, 6]
    let total = sum(my_array)
    let extended_array = push(my_array, 8)
    let new_total = sum(extended_array)
    return (total, new_total)
}
```

There are a few things to consider about what's really going on when this runs. First: in the call `sum(my_array)`, is `my_array` passed by copy or by direct reference? Logically, since `sum` has no reason to need to make a mutated copy of the array, it should be passed by reference.

Second: in the call `sum(my_Array)`, is `my_array` passed by copy or by direct reference? We have three possibilities:

* The array is passed by reference, and then a mutated copy is created by `push` and returned.
* The array is passed as a copy, and `push` internally adds the element to the copy and returns it.
* The array is passed directly by reference because `my_array` is never referenced past that point, so `push` can internally mutate the original.

While this may seem to violate the spirit of functional programming, since a value is being changed, it's important to note that this is just an *implementation detail* for optimization. From the programmer's perspective, there is no need to ever worry about a collection changing - the compiler will only do this if it is not referenced past that point.

If the collection *is* referenced past that point, the compiler would need to implicitly copy the collection. In other words, `push` needs an *exclusive* (safely mutable) reference while `sum` only needs a *shared* (immutable) reference.

What about a function that creates and returns a collection, though?

```
fn create_array(): array<int> {
    let my_array = [2, 4, 6, 8]
    return my_array
}
```

In this case, a regular reference type won't work because the data would be stack-allocated and would be gone after the function returns, and a value type on its own won't work because the size of `array` can't be known at compile-time. For this, we can use a static reference as long as the array never needs to be mutable because the data is a literal. If, however, the returned array is expected to be mutable, or if the data is not from a literal, we need a heap reference.

```
fn create_data(): array<int> {
    let my_array = range(1, 5) * 2
    return my_array
}
```

In this case, `range` internally allocates a heap reference to an array and returns it, which implicitly transfers ownership of the data to the `create_data` scope. The multiplication is implicitly an operation that takes an exclusive mutable reference to the array and multiplies the elements in-place, which it can do because the unmultiplied elements are not used past that point. By returning the result, `create_data` implicitly passes ownership of the heap allocation to the caller. If the heap reference is ever dropped, it is the responsibility of its owner to deallocate the data. If the unmultiplied elements were used after that, like this...

```
fn difference_of_sums(): array<int> {
    let series = range(1, 5)
    let evens = series * 2
    return sum(evens) - sum(series)
}
```

...we would need to handle the multiplication differently to preserve the original array. The compiler should detect that, because the array is used again after the multiplication, it cannot consume it be passing an exclusive reference. Instead, it must implicitly copy the data and pass a reference to the copy. If the compiler decides to allocate the copy on the heap, it is effectively creating a heap reference owned by `difference_of_sums` and must deallocate it immediately after the multiplication.

If the complex type being returned has a size known at compile-time, such as a struct, the compiler will implement a hidden "out parameter" and leave the caller responsible for allocating the memory, likely on the stack, and passing an exclusive reference for the function to write to.

If a function returns a value that is effectively a mutation of one of its arguments, the compiler should implement this by treating the argument as an "out parameter" and mutating it in-place. Such an implementation means the out parameter will require an exclusive reference, so the caller will be responsible for copying if the value cannot be consumed.

# Will It Really Help?

This might seem like a strange attempt to fuse Rust-like memory management with function programming, or an oversimplification of lifetime semantics, but I think it strikes a good balance between compile-time optimization and runtime performance. It could allow functional programming to take advantage of the memory management strategies used by lower-level programming languages.