---
title: Overlapping blanket impls
description: possible with a little pattern
date: "2019-06-05 12:00:00"
---
# Problem
Rust doesn't allow multiple impls of a trait on the same type. This rule keeps resolution transparent and reliable:
```rust
trait Blanket {
    fn blanket(&self) -> &'static str;
}

impl Blanket for u8 {
    fn blanket(&self) -> &'static str {
        "impl1"
    }
}

// Compilation fails at that point
impl Blanket for u8 {
    fn blanket(&self) -> &'static str {
        "impl2"
    }
}

fn main() {
    // If compilation succeeded, what would be printed?
    println!("{}", 0u8.blanket());
}
```

It also has an ugly side effect, that for every trait there can be only 1 blanket impl:
```rust
impl <T: ToString> Blanket for T { ... }

// Compilation fails at that point
impl <T: Clone> Blanket for T { ...}
```

Compiler is completely distrustful here. What if somebody somewhere created a structure that implemented both `ToString` and `Clone`? Should such combination suddenly be forbidden? What about `String` and `u32`? This rule prevents type hierarchy from sliding into minefield of odd rules and breakages on every other dependency update.

## Specialization is not an answer
Luckily the situation is about to improve, the new [specialization](https://github.com/rust-lang/rust/issues/31844) feature is coming. It will allow `impl` blocks override others when they are "obviously more specific". The RFC lists some rules, for example that blanket `impl`s are less specific than `impl`s for concrete structures.

Unfortunately the RFC explicitly states that the situation of 2 competing blanket impls is not resolvable by this new logic as there is no clear winner.

# Solution
The idea is to make every impl identifiable by dummy generic parameter specifying implementor:
```rust
trait Blanket<I> {
    fn blanket(&self) -> &'static str;
}
```
Every implementor must follow a convention:

## Structs
The parameter should be a struct
```rust
impl Blanket<u8> for u8 {
    fn blanket(&self) -> &'static str {
        "u8"
    }
}
```

## Traits that can be made into object
The parameter should be a trait object reference:
```rust
impl<T: ToString> Blanket<&ToString> for T {
    fn blanket(&self) -> &'static str {
        "ToString"
    }
}
```
Theoretically the parameter could be a trait object (e.g. `Blanket<ToString>`), but that would require `?Sized` constraint wherever the blanket impl was used.

## Traits that can't be made into object
For each such trait there should be created a dummy trait that can be made into object:
```rust
trait CloneBlanket {}
```
The parameter should be the dummy trait object reference:
```rust
impl<T: Clone> Blanket<&CloneBlanket> for T {
    fn blanket(&self) -> &'static str {
        "Clone"
    }
}
```

The dummy trait requires all generic parameters:
```rust
trait TryIntoBlanket<T> {
    type Error;
}

impl<T, E, U> Blanket<&TryIntoBlanket<T, Error = E>> for U
where
    U: TryInto<T, Error = E>,
{
    fn blanket(&self) -> &'static str {
        "try_into"
    }
}
```

# Usage
Let's create some impls:
```rust
impl<T: ToString> Blanket<&ToString> for T {
    fn blanket(&self) -> &'static str {
        "to_string"
    }
}

impl<T: AsRef<U>, U: ?Sized> Blanket<&AsRef<U>> for T {
    fn blanket(&self) -> &'static str {
        "as_ref"
    }
}
```
The `?Sized` constraint is important, for example it makes `AsRef<str>` accepted.

## Calling method
When concrete type has a single impl of trait, it can be used normally:
```rust
assert_eq!("to_string", 1u32.blanket());
```
When type has multiple impls, one of them must be explicitly picked:
```rust
// assert_eq!("???", "str".blanket()); // Fails to compile
assert_eq!("to_string", Blanket::<&ToString>::blanket(&"str"));
assert_eq!("as_ref", Blanket::<&AsRef<str>>::blanket(&"str"));
```

## Passing implementor
Let's create a function accepting implementors of `Blanket`:
```rust
fn blanket<T, B: Blanket<T>>(blanket: B) -> &'static str {
    blanket.blanket()
}
```
As of today `blanket: impl Blanket<T>` is not a valid solution, because it forbids explicit definition of `T` by caller. It may be changed in the future, it's one of discussion points of [impl trait RFC](https://github.com/rust-lang/rust/issues/34511).

Similarly to calling method, when type has single impl, it can be passed normally:
```rust
assert_eq!("to_string", blanket(1u32));
```
When type has multiple impls, its identifier must be explicitly passed as generic parameter:
```rust
// assert_eq!("???", blanket("str")); // Fails to compile
assert_eq!("to_string", blanket::<&ToString, _>("str"));
assert_eq!("as_ref", blanket::<&AsRef<str>, _>("str"));
```

# Drawbacks
- Types can spontaneously start requiring explicit blanket impl specification. A seemingly innocent addition of a trait impl can cause addition of second blanket impl to a distant type, possibly in different crate.
- Traits and functions start exposing useless generic parameters, they sometimes must be manually filled and dummy traits may show up. Both API and code get littered.
- API users must understand this pattern or they will have bad time if their code doesn't compile on the first try
- There is no way to verify that trait with blanket impls is correctly parametrized

Of course there may be more, less obvious drawbacks. This pattern **should not** be used lightly.

# Disclaimer
This probably is a well known pattern in some circles. Unfortunately I don't believe I ever came across it in any Rust API and I never found any resource describing it. If I reinvented the wheel, it's because I couldn't find the blueprint. From now on it's here so hopefully others can use it.

{: .center}
![BEHOLD.](blanket_kitty.jpg)

{: .center}
**True master of overlapping blanket impls**
