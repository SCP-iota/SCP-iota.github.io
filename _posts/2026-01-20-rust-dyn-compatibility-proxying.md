---
layout: blog_post
category: software
title: Rust dyn-Compatibility Proxying
description: "Not all Rust traits can be used as dynamic trait objects, but that doesn't have to limit what you can do dynamically."
---

If you’re fairly familiar with Rust, you’ve probably seen and used `dyn` trait objects: references to values that implement specific traits without knowledge of the concrete underlying type. For example, `Box<dyn std::error::Error>` is a frequent go-to error type when you don’t want to mess with breaking down every possible error type that could be returned by a function. Trait objects are intended for more than convenience, though; they give you the flexibility to mix and swap out underlying implementations of a trait at runtime.

For example, a codebase that accesses a database through a layer of abstraction will likely have a trait for the overall data connection - say, `DataRoot` - that could have multiple concrete implementations for different database server protocols. If there was no expectation of being able to select or change the specific implementation of `DataRoot` at runtime, then code could pass around a `&mut impl DataRoot` and avoid the need for dynamic trait objects. However, if, say, the application takes a command-line argument that selects between different database backends, and can even load plugins that can register new database backend types, then the application’s codebase cannot rely on the concrete implementation of `DataRoot` being known at compile-time; instead, all it can know is that the type implements `DataRoot` and the specifics are only determined at runtime. That is precisely what `dyn` trait objects are meant for, so ideally, the codebase could pass around a `&mut dyn DataRoot`.

The problem arises when `DataRoot` needs to use features that prevent it from being `dyn`-compatible. `dyn`-compatibility refers to whether a trait can be represented as a vtable, which is required for using it as a trait object, since `dyn` references internally hold a vtable pointer used for dispatching method calls. There are various things that can prevent a trait from being `dyn`-compatible, but in this case, we’ll focus on one particular common reason: a trait is not `dyn`-compatible if any of its methods have generic type parameters.

Practically speaking, that means we can’t use `impl ...` for parameters or return types without breaking `dyn`-compatibility, since such parameters are just syntax sugar for having generic type parameters for the concrete types being passed. Going back to the database example, suppose `DataRoot` has methods that return data access objects for specific types of resources. For example, say we have a `User` trait that represents access to a user record, and `DataRoot` has a `get_user` method to get a User by ID. If we didn’t need to concern ourselves with `dyn`-compatibility, that would likely look something like this:

```rust
trait User {
    fn id(&self) -> u64;
    // ...
}

trait DataRoot {
    fn get_user(&mut self, id: u64) -> impl User;
    // ...
}
```

(For those wondering why `get_user` takes a mutable `self` - that’s because, if the implementation involves communicating with a backend server, the connection object likely requires mutable access to write to the socket. For now, we’re assuming this method blocks, but realistically, it should be `async`; I’m saving `Future`s for a, well... future, installment of this topic.)

This would work if the concrete type implementing `DataRoot` were known at compile-time, but we can’t create a `dyn DataRoot` because it has a method that breaks `dyn`-compatibility: `get_user` returns a `impl ...` type, which is just a generic type parameter under the hood. Alternatively, we could make it `dyn`-compatible by changing the return type to `Box<dyn DataRoot>`, but that would mean using dynamic trait objects even in code where the concrete implementation could reasonably be known at compile-time. That’s not ideal, since we’d lose optimizations like inlining, making zero-cost abstractions impossible with this setup. The goal is to devise a way for the concrete type to be known at compile-time whenever possible, allowing more optimizations, while also providing a way to use the trait dynamically when the implementation can only be known at runtime. The only apparent way forward is to have a separate version of the trait that is kept `dyn`-compatible:

```rust
trait DataRoot<'l> {
    fn get_user(&mut self, id: u64) -> impl User + 'l;
    // ...
}

trait DataRootDyn<'l> {
    fn get_user_dyn(&mut self, id: u64) -> Box<dyn User + 'l>;
    // ...
}

impl<'l, T: DataRoot<'l>> DataRootDyn<'l> for T {
    fn get_user_dyn(&mut self, id: u64) -> Box<dyn User + 'l> {
        Box::new(self.get_user(id))
    }
}
```

Here, we give a blanket implementation of `DataRootDyn` for any type that implements `DataRoot`, allowing us to simply implement `DataRoot` for a type and get a corresponding `DataRootDyn` implementation for free. The second half of this will allow the holder of a `dyn DataRootDyn` to call its methods as if it were a regular `DataRoot`, since we don’t want the dynamic separatism to be contagious throughout the codebase and require a second version of everything. We can do this by implementing `DataRoot` for the dynamic trait objects:

```rust
impl<'l> DataRoot<'l> for dyn DataRootDyn<'l> + 'l {
    fn get_user(&mut self, id: u64) -> impl User + 'l {
        self.get_user_dyn(id)
    }
}

impl<'l, T: AsMut<dyn DataRootDyn<'l> + 'l>> DataRoot<'l> for T {
    fn get_user(&mut self, id: u64) -> impl User + 'l {
        self.as_mut().get_user_dyn(id)
    }
}
```

We implement it for both the plain dynamic type and any types that implement `AsMut` for the dynamic type, so that we can call `DataRoot`’s methods on smart pointers like `Box<dyn DataRootDyn>`. For this to work, though, we also need the returned trait, `User`, to be implemented for its own `dyn` reference types, so we’ll do something similar for it:

```rust
impl<'l, T: AsRef<dyn User + 'l>> User for T {
    fn id(&self) -> u64 {
        self.as_ref().id()
    }
}
```

It is now possible to seamlessly pass around a `dyn DataRootDyn` and use it as if it were a regular `DataRoot`:

```rust
struct FauxUser(u64);

impl User for FauxUser {
    fn id(&self) -> u64 {
        self.0
    }
}

struct FauxDataRoot;

impl<'l> DataRoot<'l> for FauxDataRoot {
    fn get_user(&mut self, id: u64) -> impl User + 'l {
        FauxUser(id)
    }
}

fn use_data_root<'l>(root: &'l mut dyn DataRootDyn<'l>) {
    println!("{}", root.get_user(42).id());
}

fn main() {
    let mut root = FauxDataRoot;
    use_data_root(&mut root);
}
```