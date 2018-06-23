+++
title = "Bitflags 1.0.1 - Part 2"
date = 2017-11-24T15:53:27-08:00
tags = ["linebyline", "crate", "rust"]
draft = false
authors = ["rhay"]
aliases = [
  "post/bitflags-part-2/"
]
+++
This is part two in the series of breaking down the rust crate `Bitflags`. If you haven't already,
go read [part one]({{% ref "post/code/rust/bitflags-part-1.md" %}})

Let's look at what `__impl_bitflags!` does.

```rust
macro_rules! __impl_bitflags {
    (
        // Pattern matching omitted
    ) => {
        impl $crate::_core::fmt::Debug for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
```

First up, it implements the ability to debug/format the struct into a nice enough string,
but first, as explained in the
[comment in the original code](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L402,409),
the code needs to do some convoluted things to
handle flags that are omitted conditionally, so let's walk through that.

```rust
#[allow(non_snake_case)]
trait __BitFlags {
    $(
        #[inline]
        fn $Flag(&self) -> bool { false }
    )+
}
```

First they create an unconditional function for the flag that converts it to a bool,
and always returns false. This is done as a trait so they can "override" it later
with a full implementation for the ones that will be included.

The code is pretty straight forward, it's a trait called `__BitFlags` that gets generated with
one function per flag. This is used later for printing out the flags.

```rust
impl __BitFlags for $BitFlags {
    $(
        __impl_bitflags! {
            #[allow(deprecated)]
            #[inline]
            $(? #[$attr $($args)*])*
            fn $Flag(&self) -> bool {
                self.bits & Self::$Flag.bits == Self::$Flag.bits
            }
        }
    )+
}
```

Next, they create an implementation for the new `__BitFlags` trait, and override
the implementation of the conversion function for each flag.  This respects the conditional
compile attributes because of the expansion of `$(? #[$attr $($args)*])*`

```rust                
        let mut first = true;
        $(
            if <$BitFlags as __BitFlags>::$Flag(self) {
```

Here we are generating a conditional that calls the flag function in the `__BitFlags` trait. This works even for ones
that were conditionally compiled out because the trait has a default implementation of the flag function which
always returns `false`.

```rust            
if !first {
    try!(f.write_str(" | "));
}
first = false;
```

This bit is a standard way of handling separators when outputting lists. If this is the first time it is outputting
something, it skips the separator, otherwise it prints it out, and then clears the `first` marker to say that we 
are no longer on the first item.

```rust                
try!(f.write_str(stringify!($Flag)));
```

[`try!`](https://doc.rust-lang.org/1.9.0/std/macro.try!.html) is a common macro for rust that unwraps a `Result`
erroring out early if necessary. In this case, we are wrapping `write_str` because i/o can always fail, and we
need a way of asserting that it passed. Keep in mind that this can only be used in a function that returns `Result`
otherwise, it'll give you confusing error messages.

[`stringify!`](https://doc.rust-lang.org/1.1.0/std/macro.stringify!.html) was a new macro for me when I started looking
into this, but what it does is takes a symbol (or any sequence of tokens) and returns a `&'static str` that is the 
text representation of the symbol. In this case, it's used to convert the `ident` version of the flag into a string for
output to `f` via `write_str`.

```rust                
            }
        )+
        if first {
            try!(f.write_str("(empty)"));
        }
        Ok(())
    }
}
```

The rest of the `fmt` function is clean up for edge cases, and returning `Ok()` once we are done. Then we get the
implementation of `Binary`, `Octal`, `LowerHex`, and `UpperHex` for covering different output formats.  For more 
information on those, check out the [fmt](https://doc.rust-lang.org/1.1.0/std/fmt/index.html) docs and the traits
defined therein.

I'll next take on each small chunk of the implementation.

```rust
#[allow(dead_code)]
impl $BitFlags {
    $(
        $(#[$attr $($args)*])*
        pub const $Flag: $BitFlags = $BitFlags { bits: $value };
    )+
```

This first bit is a serious of constants for referring to the values of the bitflag. Under the hood, these are all of
the type of the `$BitFlags` struct that the macro created, and the `bits` is initialized to the value of the flag.

```rust    
    /// Returns an empty set of flags.
    #[inline]
    pub fn empty() -> $BitFlags {
        $BitFlags { bits: 0 }
    }
```

Next we create a constructor function that returns an empty `$BitFlags` struct with all flags set to false (e.g. 
`bits = 0`)

```rust    
    /// Returns the set containing all flags.
    #[inline]
    pub fn all() -> $BitFlags {
        // See `Debug::fmt` for why this approach is taken.
        #[allow(non_snake_case)]
        trait __BitFlags {
            $(
                #[inline]
                fn $Flag() -> $T { 0 }
            )+
        }
        impl __BitFlags for $BitFlags {
            $(
                __impl_bitflags! {
                    #[allow(deprecated)]
                    #[inline]
                    $(? #[$attr $($args)*])*
                    fn $Flag() -> $T { Self::$Flag.bits }
                }
            )+
        }
        $BitFlags { bits: $(<$BitFlags as __BitFlags>::$Flag())|+ }
    }
```

The `all()` function returns a struct that has all of the flags set to true. This function takes the same approach as
`Debug::fmt` to handle conditionally compiled flags by defining a trait that defaults to false, and then only defining
implementations for flags that are conditionally compiled into the code.

Then the `all()` function returns the bit-wise OR'ing of the flags together.

The [`bits()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L509)
function returns the raw value of the bits field in the struct, and isn't worth covering in more depth.

```rust    
    /// Convert from underlying bit representation, unless that
    /// representation contains bits that do not correspond to a flag.
    #[inline]
    pub fn from_bits(bits: $T) -> $crate::_core::option::Option<$BitFlags> {
        if (bits & !$BitFlags::all().bits()) == 0 {
            $crate::_core::option::Option::Some($BitFlags { bits: bits })
        } else {
            $crate::_core::option::Option::None
        }
    }
    
    /// Convert from underlying bit representation, dropping any bits
    /// that do not correspond to flags.
    #[inline]
    pub fn from_bits_truncate(bits: $T) -> $BitFlags {
        $BitFlags { bits: bits } & $BitFlags::all()
    }    
```

`from_bits()` is a clever little function that takes a value of whatever the type of the BitFlags is (e.g. `u32`),
compares it to the return value of the `all()` function by bit-wise NOT and bit-wise AND to ensure that all of the
on values in the incoming bits value are associated with a flag in the bit field.  This returns an `Option<>` for
the bit flag, but only if it can be converted fully.

The next function `from_bits_truncate()` ignores any unassociated bits, dropping them as needed. Since this can never
fail, it returns a new bit flag value unconditionally.

I've skipped the
[`is_empty()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L533),
[`is_all()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L539),
[`intersects()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L545),
[`contains()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L551),
[`insert()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L557),
[`remove()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L563),
[`toggle()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L569),
and [`set()`](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L575)
functions because they are exactly what you'd expect from bit manipulation, but of course, for any of these to operate
correctly, they need to implement many of the [`core::ops`](https://doc.rust-lang.org/core/ops/) methods for the
bit-wise operations. Most of these are similar to the standard bit manipulation operators, except that it works on the
`bits` member of the struct. I won't disect them either, but you can view the code starting on
[line 584](https://github.com/rust-lang-nursery/bitflags/blob/f2ccedd7e1426b2ea1f0329c1d1db18b4436e9d5/src/lib.rs#L584)

```rust   
impl $crate::_core::iter::Extend<$BitFlags> for $BitFlags {
    fn extend<T: $crate::_core::iter::IntoIterator<Item=$BitFlags>>(&mut self, iterator: T) {
        for item in iterator {
            self.insert(item)
        }
    }
}

impl $crate::_core::iter::FromIterator<$BitFlags> for $BitFlags {
    fn from_iter<T: $crate::_core::iter::IntoIterator<Item=$BitFlags>>(iterator: T) -> $BitFlags {
        let mut result = Self::empty();
        result.extend(iterator);
        result
    }
}
```

One more bit of essentially boilerplate implementation that is necessary and that's the `FromIterator` and
`Extend` traits that allow the user to create values from an iterator. For the `Extend`, they call
`$BitFlag.insert()` on each value in the iterator. `extend` is attached to a value of type `$BitFlags`, so it mutates
the current value into a new value with the additional items in the iterator.

`FromIterator::from_iter` is a constructor function that returns a new instance of the `$BitFlags` struct with the 
values set from the iterator. It does this by calling extend for each value of the iterator and returning the resulting
mutated object. This is a great example of using mutable state within a function but having it return an immutable
value afterwards.

This last bit of the macro... holy crap is going to take quite a bit to understand, but I'm going to save it
for part three.  Until next week (? or whenever I decide to sit down and figure this out)

Continue on to [part 3]({{% ref "post/code/rust/bitflags-part-3.md" %}})
  
If you enjoyed this post, or if you have any feedback, corrections, or anything, please
leave a comment below.
