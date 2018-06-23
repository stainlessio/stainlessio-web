+++
title = "Bitflags 1.0.1 - Part 3"
date = 2018-01-09T20:55:00-08:00
tags = ["linebyline", "crate", "rust"]
authors = ["rhay"]
aliases = [
  "post/bitflags-part-3/"
]
+++

This is part three in the series of breaking down the rust crate `Bitflags`. If you haven't already,
go read [part one]({{% ref "post/code/rust/bitflags-part-1.md" %}}) and
[page two]({{% ref "post/code/rust/bitflags-part-2.md" %}})

And lastly, we are on to the most obtuse bit of the entire crate, which arguably I still don't fully understand, but
 let's dive in to see.  Let's start with the chunk of comment that explains what this macro does at the high level
 because it sets the context pretty well.

```rust
    // Every attribute that the user writes on a const is applied to the
    // corresponding const that we generate, but within the implementation of
    // Debug and all() we want to ignore everything but #[cfg] attributes. In
    // particular, including a #[deprecated] attribute on those items would fail
    // to compile.
    // https://github.com/rust-lang-nursery/bitflags/issues/109
    //
    // Input:
    //
    //     ? #[cfg(feature = "advanced")]
    //     ? #[deprecated(note = "Use somthing else.")]
    //     ? #[doc = r"High quality documentation."]
    //     fn f() -> i32 { /* ... */ }
    //
    // Output:
    //
    //     #[cfg(feature = "advanced")]
    //     fn f() -> i32 { /* ... */ }
```

Okay, so what it's doing is stripping out all attributes except `#[cfg]` attributes.  Let's look at how it works to do
that.

```rust
    (
        $(#[$filtered:meta])*
        ? #[cfg $($cfgargs:tt)*]
        $(? #[$rest:ident $($restargs:tt)*])*
        fn $($item:tt)*
    ) => {
```

Like all macros, this first bit is to match code in the source file.  Since the input attributes are prefixed with `?`
(and I have no idea where the `?` is introduced in the previous code), we are matching everything that is an attribute
 prefixed by `?`.  We match `#[cfg]` lines separately so we can pull those our and use them in the next bit.

```rust
        __impl_bitflags! {
            $(#[$filtered])*
            #[cfg $($cfgargs)*]
            $(? #[$rest $($restargs)*])*
            fn $($item)*
        }
    };
```

And now, we rewrite the it to wrap all that in a `__impl_bitflags!` macro, and remove the prefixed `?` from the 
`#[cfg]` attributes at the top of the definition. Now because this is a recursive macro, we are calling back into 
ourselves, and because the `#[cfg]` attribute is no longer prefixed with `?`, it will be rolled into `#[$filtered]`.

```rust
    (
        $(#[$filtered:meta])*
        // $next != `cfg`
        ? #[$next:ident $($nextargs:tt)*]
        $(? #[$rest:ident $($restargs:tt)*])*
        fn $($item:tt)*
    ) => {
```

This next match is for an attribute that is not a `#[cfg]` attribute but is prefixed with a `?`.

```rust    
        __impl_bitflags! {
            $(#[$filtered])*
            // $next filtered out
            $(? #[$rest $($restargs)*])*
            fn $($item)*
        }
    };
```

And we do another recursive call, but this time, we leave out the attribute we matched against.  This is essentially
removing the non-`#[cfg]` attributes from the resulting definition.  We keep recursing into the macro either rolling
`#[cfg]`'s into the `#[$filtered]` section, or removing them via the second match.

```rust    
    (
        $(#[$filtered:meta])*
        fn $($item:tt)*
    ) => {
        $(#[$filtered])*
        fn $($item)*
    };
}
```

And eventually, we are left with no `?` prefixed attributes, and we terminate by outputting it without recursing
into the macro.

So there we have it, bitflags crate broken-down line by line with a look at how macros can be used to make a very usable
crate.
  
If you enjoyed this post, or if you have any feedback, corrections, or anything, please
leave a comment below.
