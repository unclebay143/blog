---
title: Rust's Result type is cool
subtitle: Beyond the basics with Rust error handling
slug: rust-result-cool
tags: rust, error-handling, clean-code
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1664802897346/0RCVqNwfX.png?auto=compress
domain: naiveai.hashnode.dev
---

If you've worked with Rust before, you know how different its error
handling story is from most other languages. [The Rust Programming
Language](https://doc.rust-lang.org/book/ch09-00-error-handling.html) explains
the two primary ways of raising errors, panicking and the `Result` type, and how
you can propagate the `Result` type with the `?` operator to make recoverable
errors explicit without interfering with the happy path in a certain function.

Once you've got the hang of this system, it can cover a your needs for a long
time, especially since the `?` operator also automatically converts between
error types when there's an available `From` impl. You can even use `Box<dyn
Error>` as a sort of "supertype" error if you're using a bunch of different
functions that can error out in different ways.

Here, we're reading a configuration file from a path, that at the moment only
consists of a single integer.

```rust
fn work_with_config_file(path: &str) -> Result<i32, Box<dyn std::error::Error>> {
    // This error is an std::io::Error
    let file = std::fs::read_to_string(path)?;

    // This one is a ParseIntError
    let number = file.parse()?;

    // No From impl required!

    Ok(number)
}
```

However, we both know this isn't a particularly "proper" way of doing things.
As you progress using Rust in more advanced situations, this will begin to feel
janky. You should be take advantage of Rust's type system to create your own
`Error` types, just like the standard library itself does, so that the eventual
handler of the error far up the chain can deal with its specific type, instead
of just getting a generic boxed `Error` with their only real option being to
simply print it out for you to sit there and debug.

Unfortunately, this is hard and involves a ~~fair amount~~ [TON of
boilerplate](https://tinyurl.com/22cfbmrp) (enter that link at your own
discretion).

With more useful types of errors for a larger function, and with more data
and more useful error messages, the size of that code will get even more out
of control. While there are libraries that can implement many of these traits
automatically, it can be a lot of bloat to bring into a project, not to mention
the fact that the concerns here aren't well-separated - in this case, the
function itself should determine how to convert between the error types, not
some From impls.

I'll admit to having gotten frustrated with the whole situation and
thinking that the system is just fundamentally too much work. And while
there's much that can be improved - given there's an [entire working
group](https://github.com/rust-lang/project-error-handling/) dedicated to fixing
it - there are quite a lot of features of the standard library that not a lot of
intermediate-level Rustaceans know about that are immensely useful in reducing
some of the hassle.

## Combinators

The [documentation of the `Result`
type](https://doc.rust-lang.org/std/result/enum.Result.html) lists
an absolute treasure trove of methods that can be easy to ignore
in the world of `unwrap` and `?`, the most notable of which is
[`map_err`](https://doc.rust-lang.org/std/result/enum.Result.html#method.map_err).

```rust
fn work_with_config_file(path: &str) -> Result<i32, ConfigFileError> {
    let file = std::fs::read_to_string(path).map_err(|e| ConfigFileError::ReadError(e))?;
    let number = file.parse().map_err(|e| ConfigFileError::ParseError(e))?;

    // No From impl required, and no Box either!

    Ok(number)
}
```

You can make this even simpler by just passing in the variant name instead of a
closure to `map_err`, since it can be used as a function to create the variant,
like `map_err(ConfigFileError::ReadError)?`.

Now the function itself controls how it constructs the error types it returns,
and the error type itself simply defines how to print the error in a friendly
way. `From` impls are still appropriate if you find yourself doing this many
times or in multiple different places, but `map_err` can allow potentially
complex conversions between error types to happen inline without using `match`
and a bunch of indent levels.

But this is just the beginning.
[`and_then`](https://doc.rust-lang.org/std/result/enum.Result.html#method.and_then)
is very useful to chain fallible operations concisely; to group together
logically related operations or perform a series of small operations that would
clutter the code with `?` operators too much otherwise.

```rust
fn work_with_config_file(path: &str) -> Result<i32, ConfigFileError> {
    let number = std::fs::read_to_string(path)
        .map_err(ConfigFileError::ReadError)
        .and_then(|f| f.parse().map_err(ConfigFileError::ParseError))?;
    // only a single ?

    Ok(number)
}
```

Of course, you've got to be worried about trying to be overly concise and ending
up with a complicated line. `and_then`'s best use is to be able to easily
refactor out each fallible step into a function and make it all look clean.

```rust
fn read_config_file(path: &str) -> Result<String, ConfigFileError> {
    std::fs::read_to_string(path).map_err(ConfigFileError::ReadError)
}

fn parse_config_file(file: String) -> Result<i32, ConfigFileError> {
    file.parse().map_err(ConfigFileError::ParseError)
}

fn work_with_config_file(path: &str) -> Result<i32, ConfigFileError> {
    // No ? required at all - just directly return
    // the Result, since the types match!
    read_config_file(path).and_then(parse_config_file)
    // .and_then(something_else).and_then(even_more_things) ....
}
```

Now the reading and parsing steps can be large and hide a lot of implementation
details while still allowing the higher level code to work with it seamlessly.[^1]

These are the most useful and common combinators, but there are a *lot* more:
[`or_else`](https://doc.rust-lang.org/std/result/enum.Result.html#method.or_else),
[`map_or_else`](https://doc.rust-lang.org/std/result/enum.Result.html#method.map_or_else),
[`as_deref`](https://doc.rust-lang.org/std/result/enum.Result.html#method.as_deref), etc.
I highly encourage you to fully peruse the documentation of `Result`, because
there's a huge variety of incredibly useful operations that can reduce or even
totally eliminate the need to manually `match` on errors.

## Result ❤️  Option

The final piece of the puzzle that fully unlocked Rust's default error handling
story for me was the fact that while the `Option<T>` type is defined separately
for convenience, conceptually, it is best understood as a special case of
`Result`, equivalent to `Result<T, ()>`.

Because of this, `Option<T>` *also* implements the same useful combinators that
a `Result<T, ()>` would have!
[`and_then`](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then),
[`map`](https://doc.rust-lang.org/std/option/enum.Option.html#method.map),
they're all here!

More importantly than even this, there are several convenience methods on both
`Option` and `Result` that allow you to convert between them seamlessly, making
them highly connected.
[`ok_or`](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or),
for instance, allows you to add context to a bare `None` that you got from
somewhere else.

```rust
fn get_first(list: Vec<i32>) -> Result<i32, String> {
    // Vec::get returns an Option<i32>.
    list.get(0).ok_or("Empty list!".to_string())
}
```

You can even use `?` on `Option` values in a function that returns `Option`,
which is a fact I really wish was more highly advertised.

## Conclusion

There are many other places to learn about the extensive error handling story
in Rust, written by people much better at programming than I. I especially
encourage you to read
[BurntSushi's error handling guide](https://blog.burntsushi.net/rust-error-handling)
once you run into the limits of these combinators and want to take an even
deeper dive. But I hope that these examples brought your attention to the huge
amount of stuff available in the Rust standard library that makes writing
simple yet robust Rust code easier than it may appear while first learning. It
certainly opened up a whole new world to me when I first learned about them!

[^1]: If you're a functional programming nerd, you might recognize these as kind
of like monads. They are, but if you understand those, why are you reading this
article? Go cure cancer or something.
