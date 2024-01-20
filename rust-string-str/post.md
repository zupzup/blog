*This post was originally posted on the [LogRocket](https://blog.logrocket.com/understanding-rust-string-str/) blog on 20.07.2022 and was cross-posted here by the author.*

Depending on your programming background before encountering Rust, you might have been surprised by the fact that Rust has more than one type for strings, namely `String` and `str`.

In this post, we will look at the difference between `String` and `str`, or more specifically, between `String` / `&String` and `&str` — especially when it comes to the question of when to use which one.

Understanding the concepts touched on in this post will enable you to use strings in Rust in a more efficient way, but also to understand other people’s code better, because you will have a better idea of the thought process behind why certain decisions regarding their handling of strings were made.

Let’s get started!

## What we will cover

First, we will look at the more theoretical aspects, such as the different structure of the types and the differences when it comes to mutability and memory location.

Then, we will look at the practical implications of these differences, talking about when to use which type and why this makes sense.

Finally, we will go over some simple code examples showcasing these different ways of using strings in Rust.

If you’re just getting started with Rust, this post might be especially useful to you, as it will explain to you why sometimes your code doesn’t compile when you’re trying to achieve something using strings and string-manipulation across your program.

However, even if you’re not entirely new to Rust, strings are of fundamental relevance to many different areas of software engineering — having at least a basic understanding of how they work in the language you’re using is of huge importance.

## Theory

Here, we’ll go over the differences between the types on the language level as well as over some of the implications that has when using them.

Generally speaking, one can say that the difference comes down to ownership and memory. One thing all string types in Rust have in common is that they’re always guaranteed to be valid UTF-8.

## String

`String` is an owned type that needs to be allocated. It has dynamic size and hence its size is unknown at compile time, since the capacity of the internal array can change at any time.

The type itself is a struct of the form:

```rust
    pub struct String {
        vec: Vec<u8>,
    }
```

Since it contains a `Vec`, we know that it has a pointer to a chunk of memory, a size, and a capacity.

The size gives us the length of the string and the capacity tells us how much long it can get before we need to reallocate. The pointer points to a contiguous char array on the heap of `capacity` length and `size` entries in it.

The great thing about `String` is that it’s very flexible in terms of usage. We can always create new, dynamic strings with it and mutate them. This comes at a cost, of course, in that we need to always allocate new memory to create them.

## What is `&String`?

The `&String` type is simply a reference to a `String`. This means that this isn’t an owned type and its size is known at compile-time, since it’s only a pointer to an actual `String`.

There isn’t much to say about `&String` that hasn’t been said about `String` already; just that since it’s not an owned type, we can pass it around, as long as the thing we’re referencing doesn’t go out of scope and we don’t need to worry about allocations.

Mostly, the implications between `String` and `&String` comes down to [references and borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html).

An interesting aspect of `&String`  is that it can be deref-coerced to `&str` by the Rust compiler. This is great in terms of API flexibility. However, this does not work the other way around:

```rust
    fn main() {
        let s = "hello_world";
        let mut mutable_string = String::from("hello");
    
        coerce_success(&mutable_string);
        coerce_fail(s);
    }
    
    fn coerce_success(data: &str) { // compiles just fine, even though we put in a &String
        println!("{}", data);
    }
    
    fn coerce_fail(data: &String) { // error - expected &String, but found &str
        println!("{}", data);
    }
```

## What is `&str`?

Finally, let’s look at `&str`. Since `&str` consists just of a pointer into memory (as well as a size), its size is known at compile time.

The memory can be on the heap, the stack, or static directly from the executable. It’s not an owned type, but rather a read-only reference to a string slice. Rust actually guarantees that while the `&str` is in scope, the underlying memory does not change, even across threads.

As mentioned above, `&String` can be coerced to `&str`, which makes `&str` a great candidate for function arguments, if mutability and ownership are not required.

Its best used when a slice (view) of a string is needed, which does not need to be changed. However, keep in mind that `&str` is just a pointer to a `str`, which has an unknown size at compile time, but is not dynamic in nature, so its capacity can’t be changed.

This is important to know, but as mentioned above, the memory the `&str` points to can not be changed while the `&str` is in existence, even by the owner of the `str`.

Let’s look at some of the practical implications of this.

## Practical implications of `&str`

Most importantly, understanding the above will make you write better code that the Rust compiler will happily accept.

But beyond that, there are some implications in terms of API design, as well as in terms of performance, to consider.

In terms of performance, you have to consider that creating a new `String` always triggers an allocation. If you can avoid additional allocations, you should always do so, as they take quite some time and will put a heavy toll on your runtime performance.

Consider a case where, within a nested loop, you always need a different part of a certain string in your program. If you created new `String` sub-strings every time, you would have to allocate memory for each of the sub-strings and create much more work than you would need to, because you could just use a `&str` string slice, a read-only view of your base `String`.

The same goes for passing around data in your application. If you pass around owned `String` instances instead of mutable borrows (`&mut String`) or read-only views of them (`&str`), a lot more allocations will be triggered and a lot more memory has to be kept around.

So in terms of performance, knowing when an allocation happens and when you don’t need one is important as you can re-use a string slice when necessary.

When it comes to API design, things become a bit more involved, since you will have certain goals with your API in mind and choosing the right types is very important for consumers of your API to be able to reach the goals you lay out before them.

For example, since `&String` can be coerced to `&str`, but not the other way around, it usually makes sense to use `&str` as a parameter type, if you just need a read-only view of a string.

Further, if a function needs to mutate a given String, there is no point in passing in a `&str`, since it will be immutable and you would have to create a `String` from it and return it afterwards. If mutation is what you need, `&mut String` is the way to go.

The same goes for cases where you need an owned string, for example for sending it to a thread, or for creating a struct that expects an owned string. In those cases, you will need to use `String` directly, since both `&String` and `&str` are borrowed types.

In any case, where ownership and mutability are important, the choice between `String` and `&str` becomes important. In most cases, the code won’t compile if you’re not using the correct one anyway, but if you don’t have a good understanding of which type has which properties and what happens during conversion from one type to the other, you might create confusing APIs which do more allocations than they need to.

Let’s look at some code examples demonstrating these implications.

## Code examples of `String` and `&str` use

The following examples show off some of the situations mentioned previously and contain suggestions on what to do in each of them.

Keep in mind, that these are isolated, contrived examples and what you should do in your code will depend on many other factors, but these examples can be used as a basic guideline.

## Constant

This is the simplest example. If you need a constant string in your application, there’s really only one good option to implement it:

```rust
    const CONST_STRING: &'static str = "some constant string";
```

`CONST_STRING` is read-only and will live statically inside your executable and be loaded to memory on execution.

## String mutation

If you have a `String` and you want to mutate it inside of a function, you can use `&mut String` for the argument:

```rust
    fn main() {
        let mut mutable_string = String::from("hello ") 
        do_some_mutation(&mut mutable_string);
        println!("{}", mutable_string); // hello add this to the end
    }
    
    fn do_some_mutation(input: &mut String) {
        input.push_str("add this to the end"); 
    }
```

This way, you can mutate your actual `String` without having to allocate a new `String`. However, it is possible that an allocation takes place in this case as well, for example, if the initial `capacity` of your `String` is too small to hold the new content.

## Owned strings

There are cases, where you need an owned string; for example, when returning a string from a function, or if you want to pass it to another thread with ownership (e.g. return value, or passed to another thread, or building at run time).

```rust
    fn main() {
        let s = "hello_world";
        println!("{}", do_something(s)); // HELLO_WORLD
    }
    
    fn do_something(input: &str) -> String {
        input.to_ascii_uppercase()
    }
```

In addition, if we need to pass something with ownership, we actually have to pass in a `String`:

```rust
    struct Owned {
        bla: String,
    }
    
    fn create_owned(other_bla: String) -> Owned {
        Owned { bla: other_bla }
    }
```

Since `Owned` needs an owned string, if we passed in a `&String`, or `&str`, we would have to convert it to a `String` inside the function, triggering an allocation, so we might as well just pass in a `String` to begin with in this case.

## Read-only parameter/slice

If we don’t mutate the string, we can just use `&str` as the argument type. This will work even with `String`, since `&String` can be deref-coerced to `&str`:

```rust
    const CONST_STRING: &'static str = "some constant string";
    
    fn main() {
        let s = "hello_world";
        let mut mutable_string = String::from("hello");
    
        print_something(&mutable_string);
        print_something(s);
        print_something(CONST_STRING);
    }
    
    fn print_something(something: &str) {
        println!("{}", something);
    }
```

As you can see, we can use `&String`, `&'static str`, and `&str` as input values for our `print_something` method.

## Structs

We can use both `String` and `&str` with structs.

The important difference is, that if a struct needs to own their data, you need to use `String`. If you use `&str`, you need to use Rust lifetimes and make sure that the struct does not outlive the borrowed string, otherwise it won’t compile.

For example, this won’t work:

```rust
    struct Owned {
        bla: String,
    }
    
    struct Borrowed<'a> {
        bla: &'a str,
    }
    
    fn create_something() -> Borrowed { // error: nothing to borrow from
        let o = Owned {
            bla: String::from("bla"),
        };
    
        let b = Borrowed { bla: &o.bla };
        b
    }
```

With this code, the Rust compiler will give us an error, that `Borrowed` contains a borrowed value, but `o`, which it was borrowed from, goes out of scope, hence when we return our `Borrowed`, the value we borrow from goes out of scope.

We can fix this by passing in the value to be borrowed:

```rust
    struct Owned {
        bla: String,
    }
    
    struct Borrowed<'a> {
        bla: &'a str,
    }
    
    fn main() {
        let o = Owned {
            bla: String::from("bla"),
        };
    
        let b = create_something(&o.bla);
    }
    
    fn create_something(other_bla: &str) -> Borrowed {
        let b = Borrowed { bla: other_bla };
        b
    }
```

This way, when we return `Borrowed`, the borrowed value is still in scope.

## Conclusion

In this post, we took a look at the difference between the Rust string types `String` and `str`, looked at what those differences are and how they should impact your Rust coding habits.

Furthermore, we looked at some code examples, showing off some of the patterns where to use which type and why.

I hope this article was useful to dispel some confusion you might have had about why your string-related code doesn’t compile in Rust, or if you’re further along in your Rust journey, to help you write better Rust code.

