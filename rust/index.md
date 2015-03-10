---
layout: page
title: A Swift guide to Rust
---

I needed to onboard to Rust recently for a project.  Here is the guide that I wanted.

# Background

I'm an advanced Swift developer but I'm still learning Rust.  So some things might be wrong.  Send me a [PR on GitHub](https://github.com/drewcrawford/drewcrawford.github.io/pulls).

Generally I believe that things can be taught best by people who still remember what it's like not to know them.  A lot of the Rust tutorials out there are written by people who have used the language for years.  So they gloss over some details that are important.

Other tutorials cover things I already know, because I know Swift (another high-performance systems language) relatively well.  And also C.  So I don't need yet another pointer tutorial.  I need to know Rust.

If you are trying to go from Rust to Swift, this guide might also be useful, since it explains one in terms of the other.

# Value Types

In Swift, we have value types (Structs/Enums) and reference types (Classes).  They are "on the same level" in the sense that, they are both fundamental types.

In Rust, value types are fundamental, and reference types are secondary.  There is no "class" in Rust.  But you can still do a surprising amount of OO programming, as we'll see.

## Enums

Enums are practically the same in both languages.  One difference is that Rust uses `match` where Swift uses `select`.  And the syntax is a little different, and Rust can do a little more.  But that's nothing to write home about.

## Structs

Rust's structs are also very similar to Swift structs.  The differences are, as best as I can tell:

1.  The `struct` itself contains only the *fields*.  The functions are contained in a separate block called `impl for StructName`.  This is a lot like ObjC's `@interface` and `@implementation` separation, but unlike ObjC they are generally contained in the same file.
2.  Right before the struct declaration you can include a set of lines like `#[derive(Clone)]`.  The presence of this line magically writes an implementation for the `Clone` trait.  More info on this later.

# Reference Types

In Swift a reference type is declared with `class`.  Rust does not really have a notion of this.  Instead, there are three "reference wrappers" that turn a value type into a reference type.

Note that in all cases, you get value semantics *by wrapper*, so this is something that the *user* of the type does, not specified by the *type itself*.

You can, technically, declare a *type alias* that works out to e.g. `Box<T>` but this is not idiomatic.  People expect the caller to decide how to use the type, and for it not to be a property of the type itself.

## Box<T>

`Box<T>` is the simplest reference wrapper.  It literally just contains a pointer to the thing.  So if I say `Box::new(5);` I have a pointer to 5.

So what are these other reference wrappers?  What is there more to do?  Well, it has to do with Rust's idea of pointers.

Rust gets serious about pointer safety, much more serious than even Swift is.  For example, Rust does not want you to have 2 mutable references to the same memory, because what if somehow two threads write data at the same time.  

A `Box` only has one *owner*.  A Rust *owner* is like a Swift strong pointer.  In Swift you can do

```swift
let o1 = NSObject()
let o2 = o1 //second strong pointer to o1
println("\(o1)")
println("\(o2)")
```

However the Rust equivalent is not allowed:

```rust
fn main() {
    let a = Box::new([5]);
    let b = a;
    println!("{:?}",a);
}
```

```
test.rs:4:21: 4:22 error: use of moved value: `a`
test.rs:4     println!("{:?}",a);
                              ^
```

The Rust semantics in this code are sort of novel.  The line

```
    let b = a;
```

causes a *move*.  Semantically, the strong pointer of our `Box` *moves* from `a` to `b`.  `a` is no longer valid after the assignment.  `b` becomes valid.  You can in fact assign it as many times as you like, but only the last one is a valid strong pointer.  In Rust we say the Box *has one owner*.

# Rc

To get closer to Swift semantics, we can use `Rc`.  `Rc` is like `Box` but it can have more than one owner.  Here is an updated sample:

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new([5]);
    let b = a.clone();
    println!("{:?}",a);
}
```

This compiles.  Important changes include the use of `clone()`.  `clone()` returns a 'new' strong pointer.  If you don't use `clone()` you will get the same error as before.  Rust does not allow implicit multiple strong pointers like Swift does, you have to opt into it.

There is another difference here, that has to do with memory management.  In Swift's case, whether reference counting is used is an implementation detail.  That is, the compiler inserts the retain/release calls.  If the number of strong pointers is small, it may omit reference counting calls, or include them, as it decides.  In Rust, `retain` is explicit:  you call `clone` on an `Rc`.  So you control when this happens, not the compiler.  (`release` is what happens when your variable goes out of scope, so that is implicit, like Swift.)

# Arc

Rust has a final reference wrapper type, called `Arc`.  It is better to assume that this is nothing like the Swift technology ARC.

It is basically the same as `Rc`, except `clone` is guarded by a Mutex (and thus is threadsafe).    Swift's retain/release system on the other hand is always threadsafe, and you cannot get a mutexless version.

You may wonder why Rust has all these types for what is essentially the same thing in Swift, and what it comes down to is performance and control.  Swift takes the view that whether or not something has multiple strong pointers is something that the compiler should figure out during an optimization pass.  And so Swift does not make you worry about such things, although there are cases where it is overly cautious and you spend a little time waiting for a mutex to unlock that is not strictly necessary.

Rust takes the view that this is the programmer's responsibility.  And so you are actually opting into multiple strong pointers, and mutexes.  That can produce faster code in some cases, although it is a lot more work.

## A word on thread safety

"Forgetting to use a Mutex" may scare some Swift programmers.  The reality however is that in Rust you cannot "just" forget a mutex and so get into a thread bug situation.  Instead, your program won't compile.  So it is not really that scary.

## A word on performance of reference and value types

It's important to distinguish between reference and value types *semantically* vs reference and value types *performance-wise*.

In C for example, when we say "pass-by-value" what we really mean is "pass-with-copy".  Whereas pass-by-reference is "pass-without-copy"  And so people pass references around in C, not necessarily because they have to for some semantic reason, but because it is fast.

However, this view does not hold true for Swift and Rust.  "pass-by-value" can be zero-copy, or it can be copy-on-write, or something like that.  Pass-by-reference is not necessarily faster than pass-by-value.  Use the semantics that make sense for your program, and do not work on the presumption that one is faster than the other.

# Traits

Traits in Rust serve the purpose of several Swift features: extensions, protocols, and inheritance.

## Traits as protocol:

```rust
pub trait MyProtocol : SuperTrait {
    fn my_func(&mut self);
}
```

You see here that we have something like a protocol here.  Structs that *conform* to our *protocol* (*implement* our *trait*) must have this function.

Also like protocols, traits can have a supertrait.  This is not very commonly done in Swift, but it is indeed possible, check out `NSObjectProtocol`.  

## Traits as extensions:

When you actually go to implement a trait, you do it in a special block:

```rust
impl MyProtocol for Struct {
    fn my_func(&mut self) {

    }
}
```

This separation between the main body code and the trait code is similar to `extension` in Swift.  

Also like `extension`, you can provide a trait impelementation for a system class.

```rust
impl<T> MyProtocol for Box<T> {
    fn my_func(&mut self) {

    }
}
```

Unlike `extension` however (and possibly a better idea), you have to opt-in to a trait at the place that it is used.  For example

```rust
fn main() {
    use MyProtocol; //opt-in to MyProtocol on all objects that have an impelementation
    Box::new(5).my_func();
}
```

This avoids some of the "monkeypatching scariness" of traditional class extensions, since you have to opt into them with a fully-qualified name at the place where they are used.

If you don't do this you will get an error, and there will be a note suggesting one or more `use` lines that will get an implementation of `my_func` that is usable.

## Traits as inheritance

Finally, traits can provide a default implementation of the function.

```rust
pub trait MyProtocol : SuperTrait {
    fn my_func(&mut self) {
        println!("Hello from trait!")
    }
}
```

(Contrawise to a Swift protocol, which cannot provide a function body.)

A struct by default will inherit this implementation, but it may instead override it.

```rust
impl<T> MyProtocol for Box<T> {
    fn my_func(&mut self) {
        println!("Hello from Box<T>");
    }
}
```

(Contrawise to a Swift class extension, which cannot be overriden.)

# Closures

The Swift and Rust conception of a closure is similar, but subtly different.

Let's take a Swift example. Somewhat surprisingly, this is a legal Swift program.

```swift
var write : (() -> ())? = nil
var read: (()->())? = nil

func test() {
    var str = "Test"
    write = {
        str = "Whatever"
        return
    }
    read = {
        println("\(str)")
        return
    }
}
test()
read!() //"Test"
write!() 
read!() //"Whatever"
```

It is sort of odd to see that our `read` and `write` closures can still access `str` even though that variable is out of scope.  If this was C, our `str` would no longer be on the stack frame--it would be gone somewhere.  In Swift however, `str` continues to exist as long as some closure still exists that needs it.  It is not tied to some particular stack frame.

Now that example in Rust:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let test = ||{
        let mut str = "Test";
        write = Some(||{
            str = "Whatever";
            return
        });
        read = Some(||{
            println!("{:?}",str);
            return
        });
    };
    test();
    read.unwrap()();
    write.unwrap()();
    read.unwrap()();
}
```

Notable changes include:

1.  We don't specify a type for `read` and `write`, the compiler figures it out
2.  test is now a closure instead of a function.  In Swift, functions are simply named closures.  In Rust, they are different.  Only closures can capture the environment.

Unfortunately, the Rust compiler takes a dim view of this code, producing a full 85 lines of errors:

```
test.rs:17:5: 17:9 error: cannot borrow immutable local variable `test` as mutable
test.rs:17     test();
               ^~~~
test.rs:18:5: 18:9 error: cannot move out of `read` because it is borrowed
test.rs:18     read.unwrap()();
               ^~~~
test.rs:6:16: 16:6 note: borrow of `read` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:19:5: 19:10 error: cannot move out of `write` because it is borrowed
test.rs:19     write.unwrap()();
               ^~~~~
test.rs:6:16: 16:6 note: borrow of `write` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:20:5: 20:9 error: cannot move out of `read` because it is borrowed
test.rs:20     read.unwrap()();
               ^~~~
test.rs:6:16: 16:6 note: borrow of `read` occurs here
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:20:5: 20:9 error: use of moved value: `read`
test.rs:20     read.unwrap()();
               ^~~~
test.rs:18:5: 18:9 note: `read` moved here because it has type `core::option::Option<[closure(())]>`, which is non-copyable
test.rs:18     read.unwrap()();
               ^~~~
test.rs:8:22: 11:10 error: `str` does not live long enough
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:3:26: 21:2 note: reference must be valid for the block suffix following statement 0 at 3:25...
test.rs:3     let mut write  = None;
test.rs:4     let mut read = None;
test.rs:5
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
          ...
test.rs:7:29: 16:6 note: ...but borrowed value is only valid for the block suffix following statement 0 at 7:28
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:12         read = Some(||{
           ...
test.rs:12:21: 15:10 error: `str` does not live long enough
test.rs:12         read = Some(||{
test.rs:13             println!("{:?}",str);
test.rs:14             return
test.rs:15         });
test.rs:4:24: 21:2 note: reference must be valid for the block suffix following statement 1 at 4:23...
test.rs:4     let mut read = None;
test.rs:5
test.rs:6     let test = ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
          ...
test.rs:7:29: 16:6 note: ...but borrowed value is only valid for the block suffix following statement 0 at 7:28
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
test.rs:12         read = Some(||{
           ...
error: aborting due to 7 previous errors
```

Where do we even start?

Well, the first place we start is making test mutable, to avoid the error `test.rs:17:5: 17:9 error: cannot borrow immutable local variable 'test' as mutable`.

The second thing we do is to solve this 'str does not live long enough' business.

When a closure captures something, it can either do it "by borrow" or "by move".

1.  "by borrow" means essentially that the closure gets a reference to the enclosing stack frame.  However, the `read` and `write` functions are used outside the stack frame of `test`, so that's no good.  This is essentially what the `str does not live long enough` is about.  
2.  "by move" means that the values that are captured are *moved into* the closure.  You will recall earlier that *moving* means that the old references are invalid; that is also true of this situation.

With these changes in hand, we have a new attempt:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });
    };
    test();
    read.unwrap()();
    write.unwrap()();
    read.unwrap()();
}
```

This also fails, but with less errors.  We're making progress:

```
test.rs:18:5: 18:9 error: use of moved value: `read`
test.rs:18     read.unwrap()();
               ^~~~
test.rs:6:25: 16:6 note: `read` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
test.rs:19:5: 19:10 error: use of moved value: `write`
test.rs:19     write.unwrap()();
               ^~~~~
test.rs:6:25: 16:6 note: `write` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
test.rs:20:5: 20:9 error: use of moved value: `read`
test.rs:20     read.unwrap()();
               ^~~~
test.rs:6:25: 16:6 note: `read` moved into closure environment here because it has type `[closure(())]`, which is non-copyable
test.rs:6     let mut test = move ||{
test.rs:7         let mut str = "Test";
test.rs:8         write = Some(move ||{
test.rs:9             str = "Whatever";
test.rs:10             return
test.rs:11         });
           ...
test.rs:16:6: 16:6 help: perhaps you meant to use `clone()`?
error: aborting due to 3 previous errors
```

The difficulty here is that with `read` and `write` now `move`d into our `test` closure, they can't be used from outside it, after the closure has been declared.

So we'll move it inside.  It doesn't make a semantic difference:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });
        read.unwrap()();
        write.unwrap()();
        read.unwrap()();
    };
    test();

}
```

```
test.rs:18:9: 18:13 error: use of moved value: `read`
test.rs:18         read.unwrap()();
                   ^~~~
test.rs:16:9: 16:13 note: `read` moved here because it has type `core::option::Option<[closure(())]>`, which is non-copyable
test.rs:16         read.unwrap()();
                   ^~~~
error: aborting due to previous error
```

Now we're down to just one error, although this requires some explaining.  For now, understand that calling `.unwrap()` on an `Option` incurs a `move`.  (This is because it's declared `self`, see the section on Functions below.) So we cannot unwrap twice.

However we can unwrap into a temporary variable, and now it compiles fine:

```rust
fn main() {

    let mut write  = None;
    let mut read = None;

    let mut test = move ||{
        let mut str = "Test";
        write = Some(move ||{
            str = "Whatever";
            return
        });
        read = Some(move ||{
            println!("{:?}",str);
            return
        });

        let r = read.unwrap();
        r();
        write.unwrap()();
        r();
    };
    test();

}
```

Although the output may be unexpected:

```
"Test"
"Test"
```

This is because **moving closures in Rust have different environments**.  A moving closure essentially works from a cloned environment, frozen in time exactly as it was when the closure was declared.

Swift closures, in contrast, have shared environments.  Making changes to the environment in one closure may effect another closure that shares the environment.

Rust's non-moving closures technically have a shared environment, but it is hard to notice, because the borrow checker makes it difficult for 2 closures to read and write to the same parts of the environment.  As we saw in this example.


# Arrays

Rust's arrays are of fixed size, and the array is in fact part of the array's type.  A mutable array in Rust means that the elements can be swapped.  But elements cannot be appended, because the array size is fixed.

Rust's `Vec` type is dynamically-sized and therefore is closer to the Swift array.

# Preprocessor

Rust has a preprocessor that is stupidly powerful (*way* more powerful than C's), whereas Swift has none.  Read the Rust book for more details on `macro_rules!`, the preprocessor function.

# Functions (self, &, &mut)

A function can be defined in 3 ways.

## self

```rust
fn moving(self);
```

is a *moving* function.  This means that the ownership of the thing is transferred to the function.  e.g.

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    f.moving();
    println!("{:?}",f); //error: use of moved value 'f'.
}
```

On the plus side however, the function can assume nobody else has a reference to it.  This is very useful in some circumstances.  

### Examples

1.  If you want to ensure a function cannot be called twice (for example: `free` or `release`).  In Rust, this can be forced at compile time by using a moving function.
2.  If you want to take something that is not entirely threadsafe and use it on a different thread.  Since you know there are no other references, you can move it onto a thread without worry that somebody else might accidentally use it.

Most of the time however the moving function is too restrictive.  Instead you probably want:

## Borrow

We also write a function that `borrows` self:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn borrowing(&self) {

    }
}

fn main() {
    let f = SomeType;
    f.borrowing();
    println!("{:?}",f);
}
```

This is perfectly fine.  However, a borrowed reference has some limitations:

1.  You cannot upgrade it to a move later on.  You are only borrowing the thing, not owning it.  (You can, however, make a copy, a copy that you then own, if that's your thing.)
2.  Since you are only borrowing the thing, you must "return" it at some point, generally when the function is done executing.  Importantly, Rust will not let you squirrel away a borrowed reference into a global variable or somewhere that will potentially outlive the thing you are borrowing.
    1.  You can try and convince Rust that your location will not *really* outlive the borrowed reference by specifying [lifetimes](http://doc.rust-lang.org/book/ownership.html#lifetimes) for both.  All Rust variables have a *lifetime*, but sometimes being explicit can help the compiler understand what you are doing when you are squirreling away borrowed pointers.

## Mutable borrow

Finally, there is a mutable borrow:


```rust
fn mutable_borrow(&mut self);
```

This is exactly like borrow, except the function can now mutate self, whereas earlier it was immutable.

## Overall

Generally speaking, if you're writing a function:

1.  First try borrow (e.g. `&self`)
2.  If you need mutation, try mutable borrow (`&mut self`)
3.  Failing everything else, try a moving function (`self`).  This is mostly useful when you need to call other moving functions, or you are going to do weird things with threads.

## Interactions

You may ask, for example, what would happen if you tried to call a moving function when somebody had *borrowed* `f`?  The answer is we cannot move it, if `f` is still being borrowed:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    let g = &f; //note: borrow of `f` occurs here
    f.moving(); //error: cannot move out of `f` because it is borrowed
}
```

However, if our borrow goes out of scope, we can:

```rust
#[derive(Debug)]
struct SomeType;

impl SomeType {
    fn moving(self) {

    }
}

fn main() {
    let f = SomeType;
    {
        let g = &f;
    }
    f.moving();
}
```

# Visibility

Visibility in Rust is a little different than Swift.  Swift has three visibilitiy modifiers:

* `private`, which is *file-visible*
* `internal`, which is *target*-visible.  This is also the default.
* `public`, which is *globally*-visible

In Rust however, it's a little different.  The fundamental visibility unit is a *module*.  A file is a module, and modules are recursively composeable, so they can contain other modules, and so on.  It's turtles all the way down.  Unlike Swift, where you really have file and target as fundamentally distinct organizational units.

Rust has a notion of a target, which they call a `crate`.  But it is really just "a module that you build".  So everything is modules.

As a consequence, Rust has only two visibility modifiers:

* the default, which is visible *within the current module*
* `pub`, which is exported to the parent module

There is no way to declare a function "target-visible" (crate-visible) or "globally-visible" as such.  Instead you declare it `pub`, and then you declare the module containing it `pub` in its parent module, and then that module `pub` in its parent module, and so on, until the thing has the visibility that you want.


# Properties

Rust has no concept of Swift properties.  You have fields and functions, and that's it.

# Dispatch

Swift's dispatch is largely an implementation detail, but is mostly static, unless you opt-in with `@dynamic` or `@objc`.

Rust's dispatch on the other hand is defined to be static in most cases.  This guarantees fast performance, but has some gotchas.

One case where it is dynamic is when you have something with the type of `MyTrait` but you don't know the real type, beyond the fact that it implements the trait.  For example

```rust
#[derive(Debug)]
struct SomeType;

trait MyTrait { 
    fn borrow(&self) {

    }
}

impl MyTrait for SomeType { }

fn main() {
    let st = SomeType;
    let st_as_trait : &MyTrait = &st;
    st_as_trait.borrow(); //dynamic dispatch
}
```

This has the consequence that *you cannot call a moving method on a variable that is known only to implement a trait*, which is somewhat nonintuitive.  It's because you need to know the size of the underlying type in order to call the moving function.

# Generics

Swift's generics and Rust's generics are very very similar.

One difference is that Swift's generic specialization is an implementation detail.  Whereas Rust is guaranteed to use monomorphization.

In layman's terms, this means that the Rust compiler emits different code to implement `Vec<i32>.append(2)` than for `Vec<i64>.append(2)`.  This increases the size of your program, *but* ensures that these functions are always *statically* dispatched.  That provides some performance guarantees that Swift does not.

