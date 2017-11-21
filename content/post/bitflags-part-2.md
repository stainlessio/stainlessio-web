+++
title = "Bitflags 1.0.1 - Part 2"
date = 2017-11-13T17:12:03-08:00
tags = ["linebyline", "crate"]
draft = true
authors = ["rhay"]
+++
Let's look at what 
`__impl_bitflags!` does.

```rust
macro_rules! __impl_bitflags {
    (
        // Pattern matching omitted
    ) => {
        impl $crate::_core::fmt::Debug for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
                // This convoluted approach is to handle #[cfg]-based flag
                // omission correctly. For example it needs to support:
                //
                //    #[cfg(unix)] const A: Flag = /* ... */;
                //    #[cfg(windows)] const B: Flag = /* ... */;

                // Unconditionally define a check for every flag, even disabled
                // ones.
                #[allow(non_snake_case)]
                trait __BitFlags {
                    $(
                        #[inline]
                        fn $Flag(&self) -> bool { false }
                    )+
                }

                // Conditionally override the check for just those flags that
                // are not #[cfg]ed away.
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

                let mut first = true;
                $(
                    if <$BitFlags as __BitFlags>::$Flag(self) {
                        if !first {
                            try!(f.write_str(" | "));
                        }
                        first = false;
                        try!(f.write_str(stringify!($Flag)));
                    }
                )+
                if first {
                    try!(f.write_str("(empty)"));
                }
                Ok(())
            }
        }
        impl $crate::_core::fmt::Binary for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
                $crate::_core::fmt::Binary::fmt(&self.bits, f)
            }
        }
        impl $crate::_core::fmt::Octal for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
                $crate::_core::fmt::Octal::fmt(&self.bits, f)
            }
        }
        impl $crate::_core::fmt::LowerHex for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
                $crate::_core::fmt::LowerHex::fmt(&self.bits, f)
            }
        }
        impl $crate::_core::fmt::UpperHex for $BitFlags {
            fn fmt(&self, f: &mut $crate::_core::fmt::Formatter) -> $crate::_core::fmt::Result {
                $crate::_core::fmt::UpperHex::fmt(&self.bits, f)
            }
        }

        #[allow(dead_code)]
        impl $BitFlags {
            $(
                $(#[$attr $($args)*])*
                pub const $Flag: $BitFlags = $BitFlags { bits: $value };
            )+

            /// Returns an empty set of flags.
            #[inline]
            pub fn empty() -> $BitFlags {
                $BitFlags { bits: 0 }
            }

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

            /// Returns the raw value of the flags currently stored.
            #[inline]
            pub fn bits(&self) -> $T {
                self.bits
            }

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

            /// Returns `true` if no flags are currently stored.
            #[inline]
            pub fn is_empty(&self) -> bool {
                *self == $BitFlags::empty()
            }

            /// Returns `true` if all flags are currently set.
            #[inline]
            pub fn is_all(&self) -> bool {
                *self == $BitFlags::all()
            }

            /// Returns `true` if there are flags common to both `self` and `other`.
            #[inline]
            pub fn intersects(&self, other: $BitFlags) -> bool {
                !(*self & other).is_empty()
            }

            /// Returns `true` all of the flags in `other` are contained within `self`.
            #[inline]
            pub fn contains(&self, other: $BitFlags) -> bool {
                (*self & other) == other
            }

            /// Inserts the specified flags in-place.
            #[inline]
            pub fn insert(&mut self, other: $BitFlags) {
                self.bits |= other.bits;
            }

            /// Removes the specified flags in-place.
            #[inline]
            pub fn remove(&mut self, other: $BitFlags) {
                self.bits &= !other.bits;
            }

            /// Toggles the specified flags in-place.
            #[inline]
            pub fn toggle(&mut self, other: $BitFlags) {
                self.bits ^= other.bits;
            }

            /// Inserts or removes the specified flags depending on the passed value.
            #[inline]
            pub fn set(&mut self, other: $BitFlags, value: bool) {
                if value {
                    self.insert(other);
                } else {
                    self.remove(other);
                }
            }
        }

        impl $crate::_core::ops::BitOr for $BitFlags {
            type Output = $BitFlags;

            /// Returns the union of the two sets of flags.
            #[inline]
            fn bitor(self, other: $BitFlags) -> $BitFlags {
                $BitFlags { bits: self.bits | other.bits }
            }
        }

        impl $crate::_core::ops::BitOrAssign for $BitFlags {

            /// Adds the set of flags.
            #[inline]
            fn bitor_assign(&mut self, other: $BitFlags) {
                self.bits |= other.bits;
            }
        }

        impl $crate::_core::ops::BitXor for $BitFlags {
            type Output = $BitFlags;

            /// Returns the left flags, but with all the right flags toggled.
            #[inline]
            fn bitxor(self, other: $BitFlags) -> $BitFlags {
                $BitFlags { bits: self.bits ^ other.bits }
            }
        }

        impl $crate::_core::ops::BitXorAssign for $BitFlags {

            /// Toggles the set of flags.
            #[inline]
            fn bitxor_assign(&mut self, other: $BitFlags) {
                self.bits ^= other.bits;
            }
        }

        impl $crate::_core::ops::BitAnd for $BitFlags {
            type Output = $BitFlags;

            /// Returns the intersection between the two sets of flags.
            #[inline]
            fn bitand(self, other: $BitFlags) -> $BitFlags {
                $BitFlags { bits: self.bits & other.bits }
            }
        }

        impl $crate::_core::ops::BitAndAssign for $BitFlags {

            /// Disables all flags disabled in the set.
            #[inline]
            fn bitand_assign(&mut self, other: $BitFlags) {
                self.bits &= other.bits;
            }
        }

        impl $crate::_core::ops::Sub for $BitFlags {
            type Output = $BitFlags;

            /// Returns the set difference of the two sets of flags.
            #[inline]
            fn sub(self, other: $BitFlags) -> $BitFlags {
                $BitFlags { bits: self.bits & !other.bits }
            }
        }

        impl $crate::_core::ops::SubAssign for $BitFlags {

            /// Disables all flags enabled in the set.
            #[inline]
            fn sub_assign(&mut self, other: $BitFlags) {
                self.bits &= !other.bits;
            }
        }

        impl $crate::_core::ops::Not for $BitFlags {
            type Output = $BitFlags;

            /// Returns the complement of this set of flags.
            #[inline]
            fn not(self) -> $BitFlags {
                $BitFlags { bits: !self.bits } & $BitFlags::all()
            }
        }

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
    };

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
    (
        $(#[$filtered:meta])*
        ? #[cfg $($cfgargs:tt)*]
        $(? #[$rest:ident $($restargs:tt)*])*
        fn $($item:tt)*
    ) => {
        __impl_bitflags! {
            $(#[$filtered])*
            #[cfg $($cfgargs)*]
            $(? #[$rest $($restargs)*])*
            fn $($item)*
        }
    };
    (
        $(#[$filtered:meta])*
        // $next != `cfg`
        ? #[$next:ident $($nextargs:tt)*]
        $(? #[$rest:ident $($restargs:tt)*])*
        fn $($item:tt)*
    ) => {
        __impl_bitflags! {
            $(#[$filtered])*
            // $next filtered out
            $(? #[$rest $($restargs)*])*
            fn $($item)*
        }
    };
    (
        $(#[$filtered:meta])*
        fn $($item:tt)*
    ) => {
        $(#[$filtered])*
        fn $($item)*
    };
}
```