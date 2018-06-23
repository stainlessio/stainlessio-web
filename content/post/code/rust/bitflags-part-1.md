+++
title = "Bitflags 1.0.1 - Part 1"
date = 2017-11-21T13:00:00-08:00
tags = ["linebyline", "crate", "rust"]
draft = false
authors = ["rhay"]
aliases = [
  "post/bitflags-part-1/"
]
+++
The Line by Line series will be an in-depth look at Crates in the Rust ecosystem.  It's an attempt for me to get a 
better understanding of Rust by looking at real work examples and explaining what they do.

I am far from a competent rust programmer, so if I get something wrong, please leave a comment with the corrections 
and I'll update the posts based on your feedback with appropriate edit credits.
 
The first in the series will be the [`bitflags`](https://github.com/rust-lang-nursery/bitflags/tree/1.0.1) module.  I'm
 linking to the `1.0.1` version since that was the latest version when I started writing this post.  I'll link to 
 important lines, but will also include some lines directly from the Crate to talk about it.
 
## What does the Crate do?
 
The `bitflags` crate creates C-Style bitmask fields.  You know, those things where you `OR` values together to 
configure "things".  You see this alot in old C APIs, and OS level APIs. This crate allows you to define those in a
type-safe manner and pass them to C APIs.

## The Entry Macro

The entry point into the crate is the `bitflags!` macro. This is where the magic happens.
 
```rust
#[macro_export]
macro_rules! bitflags {
```

We start off with the macro rule for the macro. Nothing special here, this is the standard way to define a macro in 
Rust, but macros in Rust are a bit weird, so the next parts take a bit of digging for me to understand. A little 
peek into the [Rust documentation on macros](https://doc.rust-lang.org/1.7.0/book/macros.html) gave me a bit to work 
with.

```rust
    (
        $(#[$outer:meta])*
        pub struct $BitFlags:ident: $T:ty {    
            $(
                $(#[$inner:ident $($args:tt)*])*
                const $Flag:ident = $value:expr;
            )+
        }
```
This first bit is for defining the contents inside of the macro, but it's a bit obtuse if you don't know what you are
 looking at, so let's break this down even further.
 
```rust
$(#[$outer:meta])*
```

If you recall how the `bitflags!` macro is used, it contains a `struct`. This bit means that you can attach 
outside attributes (ones in the form of `#[...]`) to the struct.  In this case, we are naming them `$outer` for later
 use in the generated code.
 
```rust
pub struct $BitFlags:ident: $T:ty {   
```

Next in the macro is the ability to have a struct defined inside of the invocation body. The `$BitFlags:ident` 
creates a variable and it holds an identifier, in this case, whatever you named your struct when you called the macro
. The `$T:ty` part is saying that you also need to specify a type for the bit flags that will be generated. Even 
though this isn't encoded in the macro, it should be an unsigned integer type of some sort.

The rest of the above block is similar to the first bits, and it defines the ability to specify lines like
`const A = 0b00000001;` within the struct.

```rust
    ) => {
        __bitflags! {
```

The end of the pattern matching is the beginning of what is going to be generated
from this macro. The code invokes another macro which we'll cover in a bit when we
look at the implementation.

```rust
            $(#[$outer])*
            (pub) $BitFlags: $T {
                $(
                    $(#[$inner $($args)*])*
                    $Flag = $value;
                )+
            }
        }
    };
```

The rest of this macro is the body that will be passed into the implementation macro.
It passes the attributes and any of the `const` declarations into body and apparently
stripping the const from them.

The rest of these first macros are about the same with the main differences just covering visibility and other modifiers
 on the struct.  I won't retread those bits, so let's move on to the implementation details.
 
## The Inner Workings

```rust
#[macro_export]
#[doc(hidden)]
macro_rules! __bitflags {
    (
        $(#[$outer:meta])*
        ($($vis:tt)*) $BitFlags:ident: $T:ty {
            $(
                $(#[$inner:ident $($args:tt)*])*
                $Flag:ident = $value:expr;
            )+
        }
```

Here, we are defining another pattern for the macro. This code is all very similar to the previous pattern except 
that we now include the visiblility in the pattern. I think the main reason they chose to repeat the macro pattern 
for define visibility in the public interface was to be explicit about which types of visibility are supported.  

```rust
    ) => {
        #[derive(Copy, PartialEq, Eq, Clone, PartialOrd, Ord, Hash)]
        $(#[$outer])*
        $($vis)* struct $BitFlags {
            bits: $T,
        }
```

Now we start generating some real rust code. This defines a struct that will be named `$BitFlags` with a single member
called `bits`. It also derives implementations for a bunch of useful traits that will make working with the bit flags
easier.

```rust

        __impl_bitflags! {
            $BitFlags: $T {
                $(
                    $(#[$inner $($args)*])*
                    $Flag = $value;
                )+
            }
        }
    };
}
```

And were we have another indirection into another implementation specific macro.
  The `__impl_bitflags` is a pretty big macro, so I'm breaking that out into a separate
  article. Continue on to [part 2]({{% ref "post/code/rust/bitflags-part-2.md" %}})
  
If you enjoyed this post, or if you have any feedback, corrections, or anything, please
leave a comment below.
