[![crate](https://img.shields.io/crates/v/trait_gen.svg)](https://crates.io/crates/trait-gen)
[![documentation](https://docs.rs/trait-gen/badge.svg)](https://docs.rs/trait-gen)
[![build status](https://github.com/blueglyph/trait_gen/actions/workflows/master.yml/badge.svg)](https://github.com/blueglyph/trait_gen/actions)
[![crate](https://img.shields.io/crates/l/trait_gen.svg)](https://github.com/blueglyph/trait_gen/blob/master/LICENSE-MIT)

<hr/>

<!-- TOC -->
* [The 'trait-gen' Crate](#the--trait-gen-crate)
  * [Usage](#usage)
  * [Motivation](#motivation)
  * [Examples](#examples)
  * [Legacy Format](#legacy-format)
  * [IDE Code Awareness](#ide-code-awareness)
  * [Limitations](#limitations)
* [Compatibility](#compatibility)
* [Releases](#releases)
* [License](#license)
<!-- TOC -->

<hr/>

# The 'trait-gen' Crate

This library provides an attribute macro to generate the trait implementations for a number of
types, without the need for custom declarative macros, code repetition or blanket implementations. It makes the code clearer and easier to maintain.

Here is a short example:

```rust
use trait_gen::trait_gen;

#[trait_gen(T -> u8, u16, u32, u64, u128)]
impl MyLog for T {
    fn my_log2(self) -> u32 {
        T::BITS - 1 - self.leading_zeros()
    }
}
```

The `trait_gen` attribute generates the following code by replacing `T` with the types given as arguments:

```rust
impl MyLog for u8 {
    fn my_log2(self) -> u32 {
        u8::BITS - 1 - self.leading_zeros()
    }
}
impl MyLog for u16 {
    fn my_log2(self) -> u32 {
        u16::BITS - 1 - self.leading_zeros()
    }
}
// and so on for the remaining types
```

## Usage

```rust
#[trait_gen(T -> Type1, Type2, Type3)]
impl Trait for T {
    // ...
}
```

The attribute macro successively substitutes the `T` generic type parameter with each of the following types (`Type1`, `Type2`, `Type3`) to generate all the implementations.

All [type paths](https://doc.rust-lang.org/reference/paths.html#paths-in-types) beginning with `T` in the code have this part replaced. For example, `T::default()` generates `Type1::default()`, `Type2::default()` and so on, but `super::T` is unchanged because it belongs to another scope.

The code must of course be compatible with all the types, or the compiler will trigger the relevant errors. For example `#[trait_gen(T -> u64, f64)]` cannot be applied to `let x: T = 0;` because `0` is not a valid floating-point literal.

Also, any occurrence of `${T}` in doc comments, macros and string literals are replaced by the actual type in each implementation.

Note that using the letter "`T`" is not mandatory, any type path will do. For example, `gen::Type` is fine too. But to make it easy to read and similar to a generic implementation, using short identifiers is preferred. 

## Motivation

There are several ways to generate multiple implementations:
- copy them manually
- use a declarative macro
- use a blanket implementation

The example of implementation above could be achieved with this declarative macro:

```rust
macro_rules! impl_my_log {
    ($($t:ty)*) => (
        $(impl MyLog for $t {
            fn my_log2(self) -> u32 {
                $t::BITS - 1 - self.leading_zeros()
            }
        })*
    )
}

impl_my_log! { u8 u16 u32 u64 u128 }
```

But the result is harder to read than native code, and the developer must write the macro declaration each time, its pattern, and translate a few elements like the parameters (`$t`). Moreover, IDEs can't often provide contextual help or apply refactoring in the macro code.

It's also quite annoying and unhelpful to get this result when looking for the definition of a method generated by a declarative macro:

```rust
impl_my_log! { u8 u16 u32 u64 u128 }
```

Using blanket implementations have other drawbacks.
- It forbids any other implementation, so it only works when the implementation is fine for all bound types, current and future.
- It's not always possible to find a trait that corresponds to what we need to write. The `num` crate provides a lot of help for primitives, for instance, but not everything is covered.
- Even when the operations and constants are covered by traits, it quickly requires a long list of trait bounds.

Writing the first example as a blanket implementation looks like this. Since it's a short example, there is only one bound, but instead of `T::BITS` we had to use a subterfuge that isn't good-looking:

```rust
use std::mem;
use num_traits::PrimInt;

impl<T: PrimInt> MyLog for T {
    fn my_log2(self) -> u32 {
        mem::size_of::<T>() as u32 * 8 - 1 - self.leading_zeros()
    }
}
```

## Examples

Here are a few examples of the substitutions that are supported. More can be found in the [integration tests](tests/integration.rs) of the library. 

The first example is more illustrative of what is and isn't replaced than practical:

```rust
#[trait_gen(U -> u32, i32)]
impl AddMod for U {
    fn add_mod(self, other: U, m: U) -> U {
        const U: U = 0;
        let zero = U::default();
        let offset: super::U = super::U(0);
        (self + other + U + zero + offset.0 as U) % m
    }
}
```

is expanded into (we only show the first type, `u32`):
  
-   ```rust
    impl AddMod for u32 {
        fn add_mod(self, other: u32, m: u32) -> u32 {
            const U: u32 = 0;
            let zero = u32::default();
            let offset: super::U = super::U(0);
            (self + other + U + zero + offset.0 as u32) % m
        }
    }
    // ...
    ```

This example shows the use of type arguments in generic traits:

```rust
struct Meter<U>(U);
struct Foot<U>(U);

trait GetLength<T> {
    fn length(&self) -> T;
}

#[trait_gen(U -> f32, f64)]
impl GetLength<U> for Meter<U> {
    fn length(&self) -> U {
        self.0 as U
    }
}
```

This attribute can be combined with another one to create a _cross-product generator_, implementing the trait for `Meter<f32>`, `Meter<f64`, `Foot<f32>`, `Foot<f64>`:

```rust
#[trait_gen(T -> Meter, Foot)]
#[trait_gen(U -> f32, f64)]
impl GetLength<U> for T<U> {
    fn length(&self) -> U {
        self.0 as U
    }
}
```

is expanded into this:

-   ```rust
    impl GetLength<f32> for Meter<f32> {
        fn length(&self) -> f32 { self.0 as f32 }
    }
    impl GetLength<f64> for Meter<f64> {
        fn length(&self) -> f64 { self.0 as f64 }
    }
    impl GetLength<f32> for Foot<f32> {
        fn length(&self) -> f32 { self.0 as f32 }
    }
    impl GetLength<f64> for Foot<f64> {
        fn length(&self) -> f64 { self.0 as f64 }
    }
    ```

Multisegment paths (paths with `::`) and path arguments (`<f32>`) can be used in the parameters. Here for example, `gen::U` is used to avoid any confusion with types if many single-letter types have already been defined. Also, the types `Meter` and `Foot` keep a part of their original module path (`units`): 

_Note: `inner` needn't actually exist since it's replaced._

```rust
#[trait_gen(inner::U -> units::Meter<f32>, units::Foot<f32>)]
impl Add for gen::U {
    type Output = gen::U;

    fn add(self, rhs: Self) -> Self::Output {
        gen::U(self.0 + rhs.0)
    }
}
```

More complicated types can be used like references or slices. Here for example to generate reference implementations:

```rust
#[trait_gen(T -> u8, u16, u32, u64, u128)]
impl MyLog for T {
    fn my_log2(self) -> u32 {
        T::BITS - 1 - self.leading_zeros()
    }
}

#[trait_gen(U -> u8, u16, u32, u64, u128)]
#[trait_gen(T -> &U, &mut U, Box<U>)]
impl MyLog for T {
    fn my_log2(self) -> u32 {
        MyLog::my_log2(*self)
    }
}
```

As you see, we can use the first generic parameter `U` in the second attribute arguments.

Finally, the documentation can be customized in each implementation by using `${T}`. This also works in macros and string literals:

```rust
trait Repr {
    fn text(&self) -> String;
}

#[trait_gen(T -> u32, i32, u64, i64)]
impl Repr for T {
    /// Produces a string representation for `${T}`
    fn text(&self) -> String {
        call("${T}");
        format!("${T}: {}", self)
    }
}

assert_eq!(1_u32.text(), "u32: 1");
assert_eq!(2_u64.text(), "u64: 2");
```

-   ```rust
    impl Repr for u32 {
        /// Produces a string representation for `u32`
        fn text(&self) -> String {
            call("u32");
            format!("u32: {}", self)
        }
    }
    // ...
    ```

## Legacy Format

The attribute supports a shorter format which was used in the earlier versions:

```rust
#[trait_gen(Type1, Type2, Type3)]
impl Trait for Type1 {
    // ...
}
```

Here, `Type1` is generated, then `Type2` and `Type3` are literally substituted for `Type1` to generate their implementation. This is a shortcut for the equivalent attribute with the other format:

```rust
#[trait_gen(Type1 -> Type1, Type2, Type3)]
impl Trait for type1 {
    // ...
}
```

The legacy format can be used when there is no risk of confusion, like in the example below.
All the `Meter` instances must change, it is unlikely to be mixed with `Foot` and `Mile`, so 
using an alias is unnecessary. The type to replace in the code must be in first position in
the parameter list:

```rust
use std::ops::Add;
use trait_gen::trait_gen;

pub struct Meter(f64);
pub struct Foot(f64);
pub struct Mile(f64);

#[trait_gen(Meter, Foot, Mile)]
impl Add for Meter {
    type Output = Meter;

    fn add(self, rhs: Meter) -> Self::Output {
        Self(self.0 + rhs.0)
    }
}
```

Be careful not to replace a type that must remain the same in all implementations! Consider the following example, in which the return type is always `u64`:

```rust
pub trait ToU64 {
    fn into_u64(self) -> u64;   // always returns a u64
}

#[trait_gen(u64, i64, u32, i32, u16, i16, u8, i8)]
impl ToU64 for u64 {
    fn into_u64(self) -> u64 {  // ERROR! Replaced by i64, u32, ...
        self as u64
    }
}
```

This doesn't work because `u64` happens to be the first type of the list. To prevent it, use a different "initial" type like `i64`, or use the non-legacy format.

## IDE Code Awareness

_rust-analyzer_ supports procedural macros for code awareness, so everything should be fine for
editors based on this Language Server Protocol implementation. 

For the _IntelliJ_ plugin, this is an ongoing work that can be tracked with [this issue](https://github.com/intellij-rust/intellij-rust/issues/6908). At the moment, with plugin version 0.4.190.5263-223, the IDE was behaving correctly as if the substitutions were done, and the user can see the expanded code in a popup. But it is still experimental and the feature [must be activated by the user](https://intellij-rust.github.io/2023/03/13/changelog-190.html):

> Note that attribute procedural macro expansion is disabled by default. If you want to try out, enable `org.rust.macros.proc.attr` experimental feature.
> 
> Call Help | Find Action (or press Ctrl+Shift+A) and search for Experimental features. In the Experimental Features dialog, start typing the feature's name or look for it in the list, then select or clear the checkbox.

As a work-around, you can define an alias, for example `type T = <type>;`, and type the implementation code. The IDE will provide some help for the type defined in the alias, but not for the other types. Or you can use the legacy format and benefit from some help too.

## Limitations

* The procedural macro of the `trait_gen` attribute can't handle scopes, so it doesn't support any type declaration with the same type literal as the attribute parameter. This, for instance, fails to compile because of the generic function:

  ```rust
  use num::Num;
  use trait_gen::trait_gen;
  
  trait AddMod {
      type Output;
      fn add_mod(self, rhs: Self, modulo: Self) -> Self::Output;
  }
  
  #[trait_gen(T -> u64, i64, u32, i32)]
  impl AddMod for T {
      type Output = T;
  
      fn add_mod(self, rhs: Self, modulo: Self) -> Self::Output {
          fn int_mod<T: Num> (a: T, m: T) -> T { // <== ERROR, conflicting 'T'
              a % m
          }
          int_mod(self + rhs, modulo)
      }
  }
  ```

* The generic parameter must be a [type paths](https://doc.rust-lang.org/reference/paths.html#paths-in-types), it cannot be a more complex types like references or slices. So you can use `gen::T<U> -> ...` but not `&T -> ...`.

# Compatibility

The `trait-gen` crate is tested for rustc **1.58.0** and greater, on Windows 64-bit and Linux 64/32-bit platforms.

# Releases

[RELEASES.md](RELEASES.md) keeps a log of all the releases.

# License

Licensed under [MIT license](https://choosealicense.com/licenses/mit/).
