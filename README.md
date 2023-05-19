[![crate](https://img.shields.io/crates/v/trait_gen.svg)](https://crates.io/crates/trait-gen)
[![documentation](https://docs.rs/trait-gen/badge.svg)](https://docs.rs/trait-gen)
[![build status](https://github.com/blueglyph/trait_gen/actions/workflows/master.yml/badge.svg)](https://github.com/blueglyph/trait_gen/actions)
[![crate](https://img.shields.io/crates/l/trait_gen.svg)](https://github.com/blueglyph/trait_gen/blob/master/LICENSE-MIT)

<hr/>

<!-- TOC -->
* [The 'trait-gen' Crate](#the-trait-gen-crate)
  * [Usage](#usage)
  * [Motivation](#motivation)
  * [Examples](#examples)
  * [Alternative Format](#alternative-format)
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
types, without the need for custom declarative macros, code repetition or blanket implementations. It makes the code easier to read and to maintain.

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

The attribute is placed before the pseudo-generic implementation code. The _generic argument_ is given first, followed by a right arrow (`->`) and a list of type arguments.

```rust
#[trait_gen(T -> Type1, Type2, Type3)]
impl Trait for T {
    // ...
}
```

The attribute macro successively substitutes the generic argument `T` in the code with each of the following types (`Type1`, `Type2`, `Type3`) to generate all the implementations.

All the [type paths](https://doc.rust-lang.org/reference/paths.html#paths-in-types) beginning with `T` in the code have this part replaced. For example, `T::default()` generates `Type1::default()`, `Type2::default()` and so on, but `super::T` is unchanged because it belongs to another scope.

The code must of course be compatible with all the types, or the compiler will trigger the relevant errors. For example `#[trait_gen(T -> u64, f64)]` cannot be applied to `let x: T = 0;` because `0` is not a valid floating-point literal.

Finally, any occurrence of `${T}` in doc comments, macros and string literals are replaced by the actual type in each implementation.

_Notes:_
- _Using the letter "T" is not mandatory, any type path will do. For example, `gen::Type` is fine too. But to make it easy to read and similar to a generic implementation, short upper-case identifiers are preferred._
- _Two or more attributes can be written in front of the implementation code, to generate the cross-product of their arguments._
- _`trait_gen` can be used on type implementations too._

## Motivation

There are several ways to generate multiple implementations:
- copy them manually
- use a declarative macro
- use a blanket implementation

The example of implementation above could be achieved with this **declarative macro**:

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

But it's noisy and harder to read than native code. We must also write a custom macro each time, with its declaration, pattern, and translation of a few elements like the parameters (here, `$t`). Moreover, IDEs can't often provide contextual help or apply refactoring in the macro code.

It's also quite annoying and unhelpful to get this result when we're looking for the definition of a method when it has been generated by a declarative macro:

```rust
impl_my_log! { u8 u16 u32 u64 u128 }
```

Using **blanket implementations** have other drawbacks.
- It forbids any other implementation except for types of the same crate that are not already under the blanket implementation, so it only works when the implementation can be written for all bound types, current and future.
- It's not always possible to find a trait that corresponds to what we need to write. The `num` crate provides a lot of help for primitives, for instance, but not everything is covered.
- Even when the operations and constants are covered by traits, it quickly requires a long list of trait bounds.

Writing the first example as a blanket implementation looks like this. Since it's a short example, there is only one bound, but instead of `T::BITS` we had to use a subterfuge that isn't very good-looking:

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

Here are a few examples of the substitutions that are supported. More can be found in the [integration tests](https://github.com/blueglyph/trait_gen/blob/v0.2.0/tests/integration.rs) of the library. 

The first example is more an illustration of what is and isn't replaced than a practical implementation:

```rust
#[trait_gen(U -> u32, i32, u64, i64)]
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

Multisegment paths (paths with `::`) and path arguments (`<f32>`) can be used in the arguments. Here for example, `gen::U` is used to avoid any confusion with types if many single-letter types have already been defined.

Also, `Meter` and `Foot` **must** keep the `units` module path in argument, because there wouldn't be a substitution if those paths were in the code (the type in `impl Add for units::gen::U` doesn't begin with `gen::U` and thus isn't replaced).

_Note: `gen` needn't actually exist since it's replaced._

```rust
#[trait_gen(gen::U -> units::Meter<f32>, units::Foot<f32>)]
impl Add for gen::U {
    type Output = gen::U;

    fn add(self, rhs: Self) -> Self::Output {
        gen::U(self.0 + rhs.0)
    }
}
```

More complicated types can be used, like references or slices. This example generates implementations for the immutable, mutable and boxed referenced types:

```rust
#[trait_gen(T -> u8, u16, u32, u64, u128)]
impl MyLog for T {
    fn my_log2(self) -> u32 {
        T::BITS - 1 - self.leading_zeros()
    }
}

#[trait_gen(T -> u8, u16, u32, u64, u128)]
#[trait_gen(U -> &T, &mut T, Box<T>)]
impl MyLog for U {
    fn my_log2(self) -> u32 {
        MyLog::my_log2(*self)
    }
}
```

As you see in the cross-product generator, the first generic argument `U` can be used in the second attribute arguments (the order of the attributes doesn't matter).

Finally, this example shows how the documentation and string literals can be customized in each implementation by using the `${T}` format:

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

_Note: there is no escape code to avoid the substitution; if you need `${T}` for another purpose and you don't want it to be replaced, you have to choose another generic argument, for example `U` or `my::T`._ 

## Legacy Format

The attribute used a shorter format in earlier versions, which is still supported even though it may be more confusing to read:

```rust
#[trait_gen(Type1, Type2, Type3)]
impl Trait for Type1 {
    // ...
}
```

Here, the code is generated as is for `Type1`, then `Type2` and `Type3` are substituted for `Type1` to generate their implementation. This is a shortcut for the equivalent attribute with the other format:

```rust
#[trait_gen(Type1 -> Type1, Type2, Type3)]
impl Trait for Type1 {
    // ...
}
```

The legacy format can be used when there is no risk of collision, like in the example below. All the `Meter` types must change, and it is unlikely to be mixed up with `Foot` and `Mile`. The type to replace in the code must be in first position in the argument list:

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

## Alternative Format

The following format is also supported when the following feature is enabled:

```cargo
trait-gen = { version="0.3", features=["in_format"] }
```

**<u>Warning</u>: This feature is temporary and there is no guarantee that it will be maintained.**

Here, `in` is used instead of an arrow `->` and the argument types must be between square brackets:

```rust
use trait_gen::trait_gen;

#[trait_gen(T in [u8, u16, u32, u64, u128])]
impl MyLog for T {
    fn my_log2(self) -> u32 {
        T::BITS - 1 - self.leading_zeros()
    }
}
```

## IDE Code Awareness

_rust-analyzer_ supports procedural macros for code awareness, so everything should be fine for editors based on this Language Server Protocol implementation. The nightly version of rustc may be needed, but not as default. 

For the _IntelliJ_ plugin, this is an ongoing work that can be tracked with [this issue](https://github.com/intellij-rust/intellij-rust/issues/6908). At the moment with plugin version 0.4.190.5263-223, the IDE is behaving correctly by taking the substitutions into account, and the user can examine the expanded code in a popup. But it is still experimental and the feature [must be activated manually](https://intellij-rust.github.io/2023/03/13/changelog-190.html):

> Note that attribute procedural macro expansion is disabled by default. If you want to try out, enable `org.rust.macros.proc.attr` experimental feature.
> 
> Call Help | Find Action (or press Ctrl+Shift+A) and search for Experimental features. In the Experimental Features dialog, start typing the feature's name or look for it in the list, then select or clear the checkbox.

As a work-around if you don't want to activate the feature, you can define an alias, for example `type T = Type1;`, and write the implementation code for `T`. The IDE will provide the expected help for `Type1`, but not for the other types in argument. Or you can use the legacy format.

## Limitations

* The procedural macro of the `trait_gen` attribute can't handle scopes, so it doesn't support any type declaration with the same literal as the generic argument. This, for instance, fails to compile because of the generic function:

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

* The generic argument must be a [type path](https://doc.rust-lang.org/reference/paths.html#paths-in-types), it cannot be a more complex type like a reference or a slice. So you can use `gen::T<U> -> ...` but not `&T -> ...`.

# Compatibility

The `trait-gen` crate is tested for rustc **1.58.0** and newer, on Windows 64-bit and Linux 64/32-bit platforms.

# Releases

[RELEASES.md](RELEASES.md) keeps a log of all the releases.

# License

Licensed under either [MIT License](https://choosealicense.com/licenses/mit/) or [Apache License 2.0](https://choosealicense.com/licenses/apache-2.0/), at your option.
